<!-- Trusted provenance: F-40da515dc6da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-40da515dc6da

## Purpose

F-40da515dc6da performs an ordered, non-atomic transformation of indirectly addressed storage, exposes argument storage to an entry-supplied callable, and subsequently invokes F-cdec09f5d432. If both calls return, execution reaches a terminal operation and an attempted invocation of its resulting target. The supplied representation establishes no normal return.

Application-level intent remains **documented_only**. The evidence does not establish a collection operation, object lifecycle, allocation protocol, or safe public interface.

This contract distinguishes retained values from later reads. Overlapping storage can change subsequent observations. Equal values or destinations do not eliminate repeated effects, and interrupted execution can leave only a completed prefix of the described writes.

## Interface

### Explicit arguments

| Position | Represented width | Established use |
|---|---:|---|
| First | Signed 32 bits | Base for word reads and a byte write; operand of frame-value shifts. |
| Second | Signed 32 bits | Operand of the initial byte-read address; source of an initial byte publication. |
| Third | 32 bits | No explicit use in the supplied body. |
| Fourth | 8 bits | Value of the first memory write. |
| Fifth | 32 bits | Its addressable argument storage is passed to the entry-supplied callable. |
| Sixth | 32 bits | No explicit use in the supplied body. |
| Seventh | 32 bits | Its argument storage is cleared after that callable returns. |

“No explicit use” does not establish inaccessibility through aliases or exposed storage. No conventional return value is established.

### Additional entry state

Three represented 32-bit values have address-bearing roles:

- **Entry base:** initial read/write base.
- **Entry context:** initial context address.
- **Entry link:** value published during two later preparations and base for fresh word reads.

Another entry value supplies an executable target.

Five external signed 32-bit values provide the **first seed** through **fifth seed**. Each seed source is evaluated twice during its handoff: once to retain an address-bearing value, and once to supply a redirect publication.

A separate externally sourced operand is passed to F-cdec09f5d432. Its declaration and the dependency’s parameter types are absent. Its storage width, value width, and type are therefore **documented_only**. It must not be treated as an established 32-bit cell or 32-bit argument. Its evaluation occurs after the entry-supplied callable returns.

These role names establish neither object types nor allocation boundaries. External storage locations, aliases, and stability are unknown.

### Widths and preconditions

Established indirect memory accesses have widths of 8, 16, or 32 bits. Displacements below are byte displacements. A “`0x44` word” means a 32-bit access at that displacement, not a recovered structure member. The separately sourced call operand has no established access width.

Halfword reads are signed 16-bit reads. The product-phase halfword is explicitly extended to signed 32 bits. The initial byte’s signedness and promotion into address arithmetic remain **documented_only**.

Narrow publications retain the destination-width bits. This does not establish wrapping semantics for preceding signed additions, multiplications, or shifts. Represented integer arithmetic is 32-bit after applicable promotions; wider intermediate arithmetic is not established.

Progress requires every reached address to support its specified access and every invoked target to be executable in the actual environment. No visible validation enforces these conditions. Endianness, alignment requirements, address conversion, arithmetic overflow consequences, and storage disjointness remain unresolved.

## Behavior

### Ordering and retained values

The following phases are ordered. Reads described as fresh are separate source evaluations, not substitutions of previously retained values.

Each handoff retains its seed before separately evaluating the same external source for the redirect publication. Equality of those evaluations is not guaranteed. This describes operand sourcing, not physical bus transactions or a concurrency model.

### Initial publications

The fourth argument is written at entry-context displacement `0x11` before the initial word read at first-argument displacement `0x64`. That word is retained for the first handoff’s concluding byte.

A signed halfword is then read at entry context plus entry base. Adding that signed value to the second argument selects a byte. Its promoted value supplies the **initial redirect address**, subject to the unresolved byte-promotion rule.

Before acquiring any seed, the function publishes, in order:

1. Entry-base low byte at first-argument displacement `1`.
2. Second-argument low byte at entry-base displacement `0x0d`.
3. Full entry link at entry-base displacement `0x44`.
4. Entry-base low byte at initial-redirect displacement `0x0d`.
5. Low 16 bits of the promoted initial byte at entry base plus the retained signed halfword.

Only afterward is the entry-base `0x44` word retained as the first payload. Overlapping narrow writes can make this payload differ from the entry link just published.

### Five handoffs

Each handoff has a context, redirect address, retained payload, seed source, and concluding byte. After its preparation, its common publication order is:

- Context’s own 32-bit address at context displacement `0x54`.
- Acquisition of the retained seed.
- Separately evaluated seed-source value at redirect displacement `0x44`.
- Retained payload at retained-seed displacement `0x14`.
- Concluding byte at context displacement `0x11`.

These are distinct effects. In particular, context self-publication precedes seed acquisition, and redirect publication precedes payload publication.

| Handoff | Context | Redirect | Retained payload | Concluding byte |
|---|---|---|---|---|
| First | Entry context | Initial redirect address | Post-publication entry-base `0x44` word | Low byte of initial first-argument `0x64` read |
| Second | Selected from entry-context `0x14` | Fresh entry-context `0x44` word | Selected link’s `0x44` word after preparation | Low byte of fresh first-argument `0x64` read |
| Third | Same context as second | Fresh current-context `0x44` word | Selected link’s `0x44` word after preparation | Low byte of current context shifted left once |
| Fourth | Replacement context | Fresh old-context `0x44` word | Third-seed `0x44` word after preparation | Low byte of fresh first-argument `0x64` read |
| Fifth | Another replacement context | Fresh old-context `0x44` word | Fourth-seed `0x44` word after preparation | Retained byte from another fresh first-argument `0x64` read |

**Second-handoff preparation.** The fresh first-argument `0x64` read precedes redirect selection. A signed halfword is read at entry context plus retained first seed. The first payload’s `0x74` word then supplies a selected link; afterward, entry-context `0x14` supplies the next context.

The selected link’s low byte is published at redirect displacement `0x0d`. Next, the redirect address’s low 16 bits are published at selected link plus the signed halfword. The selected link’s `0x44` word is retained only after these publications, so applicable overlap effects participate in the payload.

**Third-handoff preparation.** Redirect selection precedes a signed-halfword read at current context plus retained second seed. The second seed’s low byte is published at second-payload displacement `0x15`, followed by the full second payload at second-seed displacement `0x44`.

Only then does redirect displacement `0x74` supply the selected link. The redirect address’s low 16 bits are published at selected link plus the signed halfword, before retaining the link’s `0x44` word. This handoff does not obtain its concluding byte from first-argument displacement `0x64`.

**Fourth- and fifth-handoff preparations.** Each starts with a fresh first-argument `0x64` read, redirect selection, and a signed-halfword read at old context plus the current retained seed.

Replacement-context selection reads the word at the signed halfword’s value plus `0x14`. The old context is not an additional operand of this lookup.

After replacement selection, the shared preparation order is:

- Self-publication of the redirect address at its own `0x54` field.
- Publication of full entry link at current seed plus preceding payload.
- Publication of the redirect address at current-seed displacement `0x14`.
- A fresh entry-link `0x44` read, whose low byte is published at current-seed displacement `0x0d`.
- Publication of the redirect address’s low byte at replacement-context displacement `0x0d`.
- Publication of its low 16 bits at current seed plus the signed halfword.

The current-seed `0x44` word is retained afterward, before common handoff publications begin on the replacement context.

The fifth preparation additionally performs three writes before replacement selection: old context at preceding-payload displacement `0x54`; the retained first-argument-read byte at old-context displacement `0x11`; and retained fourth seed at redirect displacement `0x44`. That same retained byte concludes the fifth handoff on the replacement context. No second intervening first-argument `0x64` read supplies it.

### Product-addressed transformation

After the fifth handoff, the final context’s `0x44` word is freshly retained. A signed halfword at final context plus retained fifth seed is read and extended to signed 32 bits. This is the **signed factor**.

A **factor publication** is a 32-bit write of the signed factor at its own value plus `0x54`. It is not a halfword write.

Before any product-derived pointer traversal, two consecutive identical publication pairs occur. Each pair consists of a factor publication followed by final context’s low byte at fifth-seed displacement `0x11`. Both pairs are retained in the contract even when final storage contents are unchanged by repetition.

The first product is signed factor multiplied by the retained final-context `0x44` word. Beginning at that product, three successive word lookups use displacements `0x44`, `0x74`, and `0x14`.

The resulting address selects another `0x14` word. Its previous contents are retained as the **displaced value**, then replaced with final context shifted left once. A word read at displaced-value displacement `0x14`, multiplied by signed factor, determines the **second product**. The displaced value—not the newly written replacement—is the base of that read.

A factor publication precedes publication of the displaced value’s low byte at fifth-seed displacement `0x11`. The second product’s `0x74` word then supplies the **saved link** and establishes its frame copy at offset `0x108`.

Multiplying the second product itself by signed factor determines the **final product**. Another factor publication precedes publication of the saved link’s low byte at fifth-seed displacement `0x11`. The saved link is a retained memory-read result, not the final product.

### Closing memory publications

The final product’s `0x44` word is retained before the first argument’s `0x74` word is retained. The saved link is then published as a word at fifth-seed displacement `0x64`, followed by the earlier product-field word’s low byte at fifth-seed displacement `9`.

A fresh final-product `0x44` read supplies the byte written at final-product displacement `0x11`. Because this read follows both fifth-seed writes, it can differ from the earlier retained word.

A factor publication follows. The saved link’s low byte is then published at twice the final product plus `0x11`.

The last product-addressed read is a word at signed factor multiplied by final product, plus `0x44`. Only afterward does first-argument displacement `0x14` supply the **closing destination**.

The closing byte publications are ordered:

1. Retained first-argument `0x74` low byte at saved link plus `1`.
2. Last product-addressed word’s low byte at closing-destination displacement `0x11`.
3. Low byte of twice the final product at closing-destination displacement `9`.

### Frame exposure and call boundaries

The representation distinguishes these 32-bit frame locations. Their offsets do not establish an ABI, complete physical layout, or disjointness from argument storage.

| Frame offsets | Established value before the entry-supplied call |
|---|---|
| `0x10`, `0x24` | First argument shifted left four bits |
| `0x34`, `0x4c`, `0x13c`, `0x188` | First argument shifted left eight bits |
| `0x108`, `0x128`, `0x17c` | Saved link |
| `0xe4`, `0x60`, `0xa8`, `0x1b8` | Zero |

The `0x108` assignment occurs when the saved link is read. Later preparation assigns `0x10`, then `0x34`; clears `0xe4`, `0x60`, `0xa8`, and `0x1b8` in that order; then assigns `0x24`, `0x4c`, `0x128`, `0x13c`, `0x17c`, and `0x188` in that order.

F-000000000001 receives the address of fifth-argument storage and two zero arguments. Further progress requires its return.

After that return, seventh-argument storage is cleared. Frame offset `0x19c` receives `0x08400e1c`; offsets `0x100`, `0x120`, and `0x140` are then cleared in order.

F-cdec09f5d432 receives the separately sourced external operand followed by three zeros. The external operand is evaluated after F-000000000001 returns and after the intervening assignments. Its type, width, and access representation remain **documented_only**. Neither its entry value nor preservation across the first call is assumed.

If F-cdec09f5d432 returns, F-000000000003 is represented with operands zero and `0x08401154`. The resulting executable target, designated F-000000000002, is then invoked without represented explicit arguments. A non-returning annotation accompanies this sequence; its actual outcome is unresolved.

## Side effects

Established direct effects include ordered memory accesses, frame assignments, argument-storage mutation and exposure, call boundaries, and the terminal operation.

No direct allocation or release is identified. Address publication does not establish ownership transfer, pointer-retention permission, reference counting, or lifetime extension. The exposed fifth-argument address denotes storage belonging to this invocation; permission to retain it or inspect adjacent storage is unknown.

Neither ordinary callee has an established preservation contract for globals, argument storage, frame-accessible memory, or allocation state.

No visible error result, validation branch, rollback, synchronization, or atomic-update guarantee exists. Invalid accesses and non-returning dependencies can prevent later effects while earlier writes remain completed. Fault delivery, physical access granularity, and concurrent visibility are **documented_only**.

## Dependencies

Except for F-cdec09f5d432, these identifiers are document-local opaque labels, not recovered dependency identities.

| Dependency | Established boundary |
|---|---|
| F-000000000001 | Entry-supplied callable receiving fifth-argument-storage address and two zeros. |
| F-cdec09f5d432 | Receives an externally sourced operand evaluated after the first call returns, followed by three zeros. The operand’s type and width are not established. |
| F-000000000003 | Terminal operation with represented operands zero and `0x08401154`; supplies the attempted terminal target. |
| F-000000000002 | Resulting target, invoked without represented explicit arguments. |

Calling conventions, parameter types not otherwise established, preservation rules, ownership effects, and ordinary-call return behavior remain unknown.

## Acceptance examples

These are conditional static criteria, not executed tests. Each presumes execution reaches the relevant operation and all required address and arithmetic conditions hold.

1. **Initial ordering.** With fourth argument `0xa5` and initial first-argument `0x64` low byte `0x3c`, entry-context displacement `0x11` receives `0xa5` before the first handoff publishes `0x3c`. Final contents alone do not demonstrate both writes.

2. **Aliased initial payload.** If a narrow initial publication changes entry-base `0x44`, the first payload is the subsequent word-read result, not automatically entry link.

3. **Distinct seed evaluations.** If a seed source yields different values at its two represented evaluations, seed-addressed publication uses the retained first value, while redirect publication uses the second.

4. **Signed factor.** Product-phase halfword bits `0xfffe` produce signed factor `−2`, not `65534`. This establishes sign extension, not validity of negative addresses or every subsequent product.

5. **Unresolved byte promotion.** Initial byte bits `0x80` establish neither positive `128` nor negative `−128` as the address-bearing value without an additional promotion rule.

6. **Replacement lookup.** A retained signed halfword of `32` selects replacement-context storage at `32 + 0x14`, subject to address semantics. Old context is not added again.

7. **Fifth-handoff retention.** A later change to first-argument `0x64` does not replace the byte already retained for both old-context and replacement-context publications.

8. **Repeated publications.** Both initial product-phase publication pairs occur even when they leave identical final contents.

9. **Retain before replacement.** Writing twice final context into the selected `0x14` word does not change the displaced value already retained as the next read base.

10. **Fresh closing read.** If fifth-seed `0x64` overlaps final-product `0x44`, the later fresh read incorporates applicable overlap effects rather than automatically reusing the earlier word.

11. **Call interruption.** If F-000000000001 does not return, post-call clearing, external-operand evaluation, F-cdec09f5d432, and the terminal sequence are not reached.

12. **Post-call external operand.** If F-000000000001 changes state that supplies the external operand and returns, the subsequent evaluation supplies F-cdec09f5d432’s first argument. This criterion assigns no numeric width, storage width, signedness, or parameter type to that operand.

## Uncertainties

The following remain **documented_only**:

- Application intent, supported invocation protocol, and complete calling conventions.
- Initialization and interpretation of additional entry state.
- Initial byte signedness and promotion.
- Type, value width, storage width, and access representation of the separate external operand supplied to F-cdec09f5d432.
- Endianness, alignment, address conversion, and overlapping-access interpretation.
- Overflow and negative-operand behavior in signed arithmetic and shifts.
- Physical frame layout and aliases with arguments or indirectly addressed storage.
- Callee effects, preservation guarantees, ownership, and lifetime permissions.
- External-source stability, physical transaction counts, and concurrent visibility.
- Terminal-operation semantics, operand significance, and actual target reachability.
- Behavior of omitted regions described as unreachable.

These gaps do not authorize assumptions of stable globals, disjoint storage, universally wrapping arithmetic, or conventional exception handling.

## Evidence

**Established static evidence:** represented argument and indirect-access widths; signed halfword reads; address displacements and relationships; ordered publications; retained-versus-fresh distinctions; repeated writes; frame assignments; explicit call operands; argument-address exposure; post-call evaluation of the separate external operand; and the terminal non-returning annotation.

**Inferred intent:** terms such as context, seed, payload, and link group observable relationships only. They do not establish application abstractions or ownership protocols.

**Unestablished behavior:** successful execution, concrete address validity, dependency effects, physical transactions, actual termination mechanics, and the separate external operand’s declaration and parameter type. Its appearance as a call operand does not establish a 32-bit value or storage cell.

No runtime tests, execution traces, memory observations, or fault reproductions have been performed.
