<!-- Trusted provenance: F-14dea34ae2f6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-14dea34ae2f6

## Purpose

The supplied static representation describes a routine that interleaves indirect dependency calls with byte-sized and 32-bit memory writes. Values obtained from dependencies are repeatedly scaled, masked, split into word halves, and reused as addresses or stored data. Several writes target low absolute addresses or addresses selected by a byte read from another location.

A higher-level purpose is not established. In particular, the evidence does not establish that this routine allocates objects, updates a valid data structure, implements a protocol, or completes a normal operation.

The representation carries an invalid-instruction warning and ends at an invalid-data boundary. Accordingly, this contract describes the visible execution prefix, conditional on the supplied representation being accurate and each preceding operation completing. It does not assert that the entire prefix is executable on the original target.

## Interface

### Explicit inputs and output

The represented interface has two 32-bit arguments and no declared return value.

- **First argument:** No explicit use is visible.
- **Second argument**, called **Payload** below: Its low eight bits are written twice to one derived byte address. Its full 32-bit value is also written to another derived address.

The unused first argument is not sufficient evidence that its incoming machine location is irrelevant to dependencies. The representation does not fully describe the calling convention.

No normal return is shown. The visible prefix terminates at an invalid-data boundary rather than at an ordinary return.

### Additional incoming state

Three ambient inputs are material:

- **Seed:** An incoming unsigned 32-bit value.
- **Initial byte base:** An incoming address used for one byte write.
- **Initial dispatch basis:** An incoming value used to form the first indirect-call target.

These are not explicit interface arguments in the supplied representation. Their initialization and validity are unknown.

The routine additionally relies on dependency return values, a secondary machine-result value, existing memory, and the ability to execute indirect calls.

### Width and arithmetic conventions

For the descriptions below:

- A **byte write** stores the low eight bits of its value.
- A **word write** stores 32 bits.
- Right shifts of the named unsigned values are logical.
- A **pair result** means the represented 64-bit dependency result, divided into low and high 32-bit halves.
- Scaled values and addresses are described using their 32-bit machine-bit-pattern interpretation.

For example, “four times a value” denotes its low 32-bit scaled result under that interpretation. Some source expressions are typed as signed arithmetic. Their overflow behavior cannot be established as a portable language-level guarantee; exact machine arithmetic remains a documented uncertainty.

No byte order, alignment policy, address-space permissions, or memory-mapped-I/O interpretation is established.

## Behavior

### Reading the dependency identifiers

The dependency identifiers F-D001 through F-D107 below are opaque labels assigned to unresolved **call sites**. They are not verified identities of distinct functions. Multiple sites could reach the same function.

An inclusive identifier range denotes calls in ascending order. Except where arguments or results are stated, a call has no explicit argument or consumed result in the supplied representation. This does not mean that it receives no machine state or has no side effects.

All phases below are ordered. A later phase is reached only if earlier calls and memory accesses complete.

### 1. Initial indexing and publication

The routine retains Seed shifted right by four for a later byte write.

F-D001 receives Seed shifted right by two. F-D002 through F-D005 then execute; F-D005 supplies a 32-bit value called **Root index**.

A word write stores Seed shifted right by two at address:

- four times Root index, plus 20.

F-D006 receives four times Root index. F-D007 through F-D010 execute, with F-D010 producing **Descriptor value**.

The routine then performs, in order:

1. A byte write at Initial byte base plus one, storing the low byte of eight times Root index.
2. A word write at eight times Root index plus 52, storing Descriptor value shifted left by two.

These are distinct accesses even if their addresses happen to overlap.

### 2. Descriptor-derived writes

F-D011 receives Descriptor value shifted left by two. F-D012 supplies an opaque 32-bit result.

The routine retains **Descriptor base**, defined as Descriptor value with only its low 30 bits retained.

F-D013 receives the result from F-D012. F-D014 through F-D017 follow; F-D017 supplies a pair result called **First pair**.

Two writes follow:

1. A byte write at Descriptor base plus one, storing the low byte of eight times Root index.
2. A word write at eight times Root index plus 52, storing the high half of First pair.

The second write revisits the word address used in phase 1. It is not safe to remove the earlier write: dependency calls intervene and may observe or modify memory.

### 3. Root-scaled byte publication

F-D018 receives the low half of First pair shifted left by two. F-D019 through F-D022 execute; F-D022 supplies **First dispatch value**.

The routine retains **Root span**, equal to 16 times Root index, and performs:

1. A byte write at address 17 plus Descriptor base shifted right by 20, storing the low byte of Root span.
2. A byte write at Root span plus 13, storing the low byte of Payload.

The first destination lies between addresses 17 and 1040 inclusive, because Descriptor base contains at most 30 significant bits.

F-D023 receives two identical arguments, each First dispatch value shifted left by two. F-D024 supplies **Second dispatch value**. F-D025 receives that value and the same value shifted right by three.

F-D026 through F-D029 then execute; F-D029 supplies **Third dispatch value**.

The routine writes the low byte of Root span to absolute address 17, then writes the low byte of Payload again at Root span plus 13.

### 4. Payload and working-base setup

F-D030 receives Third dispatch value shifted left by two. F-D031 through F-D034 execute; F-D034 supplies **Working value**.

The routine writes:

1. Payload as a word at Descriptor base plus 20.
2. The low byte of twice Descriptor base at Descriptor base plus 13.

It retains:

- **Working base:** 32 times Working value.
- **Working word:** Working value shifted left by seven.
- **Compact index:** Working value masked to its low 27 bits, then shifted right by 17.

Compact index is between 0 and 1023 inclusive.

F-D035 through F-D037 execute; F-D037 supplies **Expansion value**.

The routine writes Working word at Working base plus 20, then writes the low byte of Expansion value shifted left by two at Working base plus 13.

### 5. Byte-indexed publication

F-D038 receives Expansion value shifted left by two. F-D039 through F-D042 execute.

The next observable memory operations are:

1. A byte write at Compact index plus seven, storing the low byte of twice the earlier retained value Seed shifted right by four.
2. A byte read at Working base plus one.
3. A byte write at the address obtained by adding 17 to that unsigned byte, storing the low byte of Compact index.

The second destination is therefore within addresses 17 through 272. The byte read is not inferred from any preceding write; it reads memory as it exists at this point.

F-D043 receives three identical arguments, each Expansion value shifted left by 21. F-D044 supplies **Retained value**. F-D045 receives Retained value. F-D046 through F-D049 execute; F-D049 supplies **Next dispatch value**.

The routine reads a byte again at Working base plus one, retaining it as **Selector**, and writes zero at Selector plus 17. This read occurs after intervening dependencies and need not return the earlier byte.

### 6. Selector-scaled state

F-D050 receives Next dispatch value shifted left by two. F-D051 through F-D054 execute.

The routine retains:

- **Selector base:** twice Selector, in the range 0 through 510.
- **Retained byte:** the low byte of Retained value shifted right by three.

It writes Retained byte at Working base plus 13, then writes Selector to absolute address 13.

F-D055 receives Selector base, 152, and 608. F-D056 executes. F-D057 then receives a secondary machine-result value shifted right by three. That secondary value is not modeled as an ordinary primary return and cannot be reliably attributed to a specific dependency from this representation alone.

F-D058 through F-D061 execute; F-D061 supplies **Conversion input**. F-D062 receives Conversion input shifted left by two and supplies **Conversion result**.

The routine then writes Retained byte at Selector base plus 13, followed by the constant byte 192 at absolute address 13.

### 7. Linked-base access

F-D063 receives Conversion result shifted left by two. F-D064 through F-D067 execute; F-D067 supplies **Link value**.

The next accesses, in order, are:

1. A word write of Link value shifted left by two at Selector base plus 36.
2. A word read at Selector base plus 116, retained as **Linked base**.

The routine also retains **Link word**, equal to Link value shifted left by five.

F-D068 receives four arguments, in order:

- Linked base.
- Linked base shifted left by one.
- Link word.
- Linked base masked to its low 31 bits, then shifted right by two.

After F-D069 through F-D072 execute, the routine writes Link word at Linked base plus 36.

It next reads a word at Linked base plus 116 and passes that freshly read value to F-D073. That read must not be replaced by an earlier assumed value.

### 8. Pair-derived links

F-D074 through F-D077 execute; F-D077 supplies **Second pair**.

The routine retains **Pair-scaled value**, equal to the low half of Second pair shifted left by two. It writes Retained value shifted right by three as a word at four times the high half of Second pair, plus 20.

F-D078 receives Pair-scaled value three times, followed by the high half of Second pair shifted right by three.

F-D079 supplies **Third pair**. F-D080 receives, in order:

- The low half of Third pair.
- The high half of Third pair.
- The low half again.
- The high half shifted right by three.

F-D081 through F-D084 execute; F-D084 supplies **Destination index**.

The routine writes Pair-scaled value at four times Destination index, plus 20. F-D085 then receives four times Destination index.

### 9. Repeated byte-indirect updates

F-D086 through F-D089 execute; F-D089 supplies **Final value**.

The routine retains **Final scaled value**, equal to Final value shifted left by two. It reads a byte at Linked base plus one and writes the low byte of Final scaled value at that byte plus 17.

F-D090 through F-D092 execute; F-D092 supplies **Penultimate dispatch value**.

The routine again reads a byte at Linked base plus one, then writes the low byte of Final value masked to 30 bits and shifted right by 20 at that byte plus 17.

The two byte reads are separate. Intervening dependency effects can make the destinations differ.

F-D093 receives Penultimate dispatch value shifted left by two. F-D094 through F-D097 execute; F-D097 supplies **Final dispatch value**.

The routine reads a byte at 1024 times Final scaled value, plus one, then writes zero at that byte plus 17.

### 10. Final visible operations

F-D098 receives, in order:

- Final scaled value shifted left by three.
- The same value shifted left by three.
- Final scaled value.
- Final dispatch value shifted left by two.

F-D099 through F-D102 execute; F-D102 supplies **Terminal dispatch value**.

The routine again reads a byte at 1024 times Final scaled value, plus one, and writes zero at that byte plus 17. This is a fresh read after intervening calls, not a reuse of the previous selector.

F-D103 receives Terminal dispatch value shifted left by two. F-D104 through F-D107 execute.

The supplied representation ends at an invalid-data boundary. No subsequent behavior or successful return is established.

## Side effects

### Memory and ordering

The visible prefix contains direct byte and word writes, direct byte and word reads, and dependency calls whose memory effects are unknown.

Byte-indirect destinations formed from an unsigned byte plus 17 always fall within addresses 17 through 272. Absolute addresses 13 and 17 are written explicitly. These accesses must not be silently reinterpreted as ordinary object fields.

Other recurring offsets include one, 13, 20, 36, 52, and 116. Those offsets do not establish a structure layout, object extent, alignment, ownership relationship, or field meaning.

Aliasing is unrestricted by the evidence:

- Different derived addresses may coincide.
- A byte write may overlap a word previously written.
- Dependencies may alter memory that later reads consult.
- Dependencies may observe intermediate writes that are subsequently overwritten.

Consequently, neither repeated writes nor repeated reads are redundant merely because their address expressions match.

### Ownership and lifetime

No allocation, release, reference acquisition, or ownership transfer is explicitly established. A dependency result used as an address is not thereby proven to reference live storage.

The caller or surrounding execution environment must provide every required executable target and accessible memory location if the prefix is to complete. This is a conditional execution prerequisite, not evidence of checks performed by the routine.

### Errors and termination

No explicit validation, error result, recovery path, or rollback appears.

An invalid indirect-call target, inaccessible address, disallowed alignment, or dependency failure may prevent later operations. The evidence does not establish the resulting exception, fault code, process behavior, or partial-effect recovery.

The final invalid-data boundary is evidence of incomplete or invalidly decoded control flow, not a documented application-level error mechanism.

## Dependencies

F-D001 through F-D107 are unresolved dependencies, with their visible argument and result relationships specified above.

F-D001 uses the incoming Initial dispatch basis to form its target. Later targets are represented using embedded instruction-location-derived bases and fixed offsets. They are not supplied through an explicit callback parameter.

The evidence does not establish:

- Concrete dependency identities or signatures.
- Whether multiple sites share a target.
- Whether the represented zero-explicit-argument calls have meaningful implicit inputs.
- Whether dependencies preserve global state or shared memory.
- Whether dependencies allocate, retain, transfer, or release resources.
- Whether dependencies return normally.
- Whether pair results represent one scalar, two independent results, or an imperfect reconstruction of machine state.

F-14dea34ae2f6 cannot therefore be specified as pure, reentrant, thread-safe, or ownership-neutral.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial scaling:** If Seed is hexadecimal `12345678` and F-D005 supplies Root index 3, the first word write targets address 32 and stores hexadecimal `048D159E`. This requires all preceding calls to complete.

2. **Payload truncation:** If Payload is hexadecimal `A1B2C3D4`, each Payload byte write stores hexadecimal `D4`. The Payload word write stores the full 32-bit value. Its byte order is unspecified.

3. **Repeated word publication:** If Root index remains the retained value 3, both writes at eight times Root index plus 52 target address 76. The first stores Descriptor value shifted left by two; the later one stores the high half of First pair. The intervening calls remain observable.

4. **Fresh selector reads:** If the first read at Working base plus one returns 5, the following byte write targets address 22. If the later read returns 9, the subsequent zero write targets address 26 instead. A conforming reconstruction cannot reuse the earlier value.

5. **Unsigned selector scaling:** If Selector is 255, Selector base is 510. Its later Retained byte write targets address 523; the link write and read target addresses 546 and 626 respectively.

6. **Compact-index boundary:** If Working value’s low 27 bits are all one, Compact index is 1023. The Seed-derived byte write targets address 1030. The byte-indirect publication stores only hexadecimal `FF`, not the full index.

7. **Dependency observation:** If a dependency observes a location between two visible writes to it, the first value is part of the contract. Collapsing those writes into the final value is not justified.

8. **Incomplete execution:** If the first absolute-address-13 write faults, no later dependency call or write is required by this prefix. No rollback or error return is established.

## Uncertainties

The following are **documented_only**:

- Whether the represented routine boundary and control flow correspond to valid executable instructions.
- The true calling convention, including ambient inputs and secondary results.
- Exact dependency targets, signatures, and side effects.
- Machine overflow semantics for expressions presented with signed types.
- Endianness, alignment requirements, and address-space mapping.
- The validity and lifetime of all derived addresses.
- Any business meaning, data-structure schema, or protocol interpretation.
- Whether execution can reach every displayed operation.
- Any continuation beyond the invalid-data boundary.

**Inferred intent:** The recurring shifts, masks, offsets, and byte-selected destinations may reflect packed state or encoded references. That interpretation is not required for acceptance and must not replace the explicit memory behavior.

## Evidence

**Established static evidence:** The supplied representation exposes two 32-bit formal arguments, three material ambient inputs, ordered indirect calls, byte and word memory operations, several pair-result uses, no explicit argument validation, and no ordinary return before an invalid-data boundary.

**Evidence limitation:** The representation itself warns of invalid instruction data. This contract preserves the visible dependencies and observable ordering without treating that representation as proof of executable validity.

**Runtime status:** No runtime tests have been performed. The acceptance examples are arithmetic and ordering consequences of the supplied static representation, subject to the stated execution and width conditions.
