<!-- Trusted provenance: F-e7ec0d081d3a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7ec0d081d3a

## Purpose

F-e7ec0d081d3a performs an overlap-sensitive transformation of addressed memory, replaces a discovered link, prepares invocation-associated storage, and transfers control to external dependencies.

Its established contract concerns access widths, address relationships, retained values, arithmetic, and ordering. An application-level purpose, public data structure, supported input domain, and normal-return result are not established.

The behavior below is conditional on valid arithmetic and accessible storage. A late byte write has no established payload initialization. Depending on the applicable execution semantics, that defect may prevent meaningful continuation or any unconditional guarantee about observable effects.

## Interface

### Inputs and result

The represented interface has seven explicit 32-bit argument positions.

| Position | Established role |
|---|---|
| First | Numeric address base, stored payload, and signed-shift operand |
| Second and third | No direct use of their incoming values is established |
| Fourth | Initial word payload and input to a character conversion |
| Fifth | Its argument-storage address is passed to a dependency |
| Sixth | No direct use of its incoming value is established |
| Seventh | Its argument storage is assigned zero if the first dependency returns |

The fifth argument is exposed by **address of storage**, not by interpreting its incoming value as an address.

Additional incoming state provides an initial base, initial recipient, donor address, back-reference, and invocation target. Their delivery mechanism is **documented_only**. The back-reference serves both as a numeric payload and an address.

No explicit return value, recoverable-error result, or ordinary returning continuation is established.

### Widths and preconditions

Words are 32 bits, halfwords 16 bits, and bytes eight bits. Displacement reads are signed halfwords promoted for subsequent arithmetic.

A character-derived address is the promoted character value itself. Character signedness and out-of-range character conversions remain unresolved. A **raw low byte**, where specified below, preserves the low eight bits without character-sign interpretation. Signed-halfword narrowing is not universally established as unsigned truncation.

Offsets are decimal byte offsets unless prefixed with `0x`. Address arithmetic is represented using signed 32-bit integers; modular wrapping is not established. Under conventional signed semantics:

- Reached additions and multiplications must have representable results.
- Signed left shifts require permitted operands and representable results.
- Subsequent narrowing does not repair an invalid shift or overflow.

Every reached access requires appropriate extent, alignment, permissions, lifetime, and access compatibility. No validation establishes those conditions. Byte order, integer-to-address interpretation, and concrete argument or invocation-storage layout are **documented_only**.

## Behavior

### Memory-routing contract

Ten routing stages distinguish successive address relationships. They are specification labels, not a public record layout or a requirement to implement a loop.

Each stage has a **base**, **recipient**, **incoming carrier**, and **selected address**. A stage-specific external word source supplies two separate observations: one selects the address; the other supplies a word written to recipient offset 68. Their values are not assumed equal.

The following retained values recur:

- **Snapshot:** a word read from first-argument offset 100 at a specified stage.
- **Peer:** a word read from the current base at offset 68.
- **Displacement:** a signed halfword read at the sum of base and selected address.
- **Route:** a character read at the sum of displacement and incoming carrier, then promoted directly to an address.
- **Central reference:** the word retained from stage 4’s selected address at offset 100.

The central reference is not a first-argument snapshot.

Before routing begins, initial-base offset 84 receives the fourth argument as a word. A halfword at twice the initial base plus donor then receives the narrowed initial recipient. Only afterward is the first carrier read from donor offset 68. Thus, overlap with either initialization write can affect that carrier. Doubling here is multiplication, not shifting.

### Stage relationships

Stages 2–10 receive the preceding stage’s closing carrier.

| Stage | Base | Recipient | Entry-byte input |
|---|---|---|---|
| 1 | Initial base | Initial recipient | Fourth argument |
| 2 | Stage-1 successor | Stage-1 peer | Stage-1 snapshot |
| 3 | Stage-2 successor | Stage-2 peer | Current base shifted left one bit |
| 4 | Base retained from stage 3 | Stage-3 route | Current base shifted left one bit |
| 5 | Base retained from stage 3 | Stage-4 route | Current base shifted left one bit |
| 6 | Base retained from stage 3 | Stage-5 route | Stage-5 snapshot |
| 7 | Stage-6 successor | Stage-6 peer | Stage-6 snapshot |
| 8 | Base retained from stage 7 | Stage-7 peer | Current base shifted left one bit |
| 9 | Stage-8 successor | Stage-8 peer | Stage-8 snapshot |
| 10 | Stage-9 successor | Stage-9 peer | Retained raw low byte of stage-9 snapshot |

Entry-byte inputs undergo character conversion except in stage 10, which preserves the retained raw byte.

### Ordering shared by routing stages

Each stage requires this ordered access boundary:

1. Base offset 84 receives the base as a word.
2. The first external-source observation determines the selected address.
3. The separate external-source observation supplies recipient offset 68.
4. Selected-address offset 20 receives the incoming carrier as a word.
5. Base offset 17 receives the entry byte.
6. The displacement is read.

Additional reads between the entry-byte write and displacement read are:

| Stages | Required intervening reads |
|---|---|
| 1, 6, 8, 9 | Snapshot, then peer |
| 2, 7, 10 | Peer |
| 5 | Snapshot |
| 3, 4 | None |

The retained values preserve what those reads observed, even if later writes overlap their sources.

Stages 1–9 finish with a halfword write followed by a fresh word read that supplies the next carrier:

| Stages | Halfword destination | Narrowed payload | Closing-carrier source |
|---|---|---|---|
| 1 | Displacement plus selected address | Peer |
Selected address, offset 68 |
| 2 | Displacement plus redirected address | Peer | Redirected address, offset 68 |
| 3–5 | Displacement plus selected address | Route | Selected address, offset 68 |
| 6–7 | Displacement plus redirected address | Peer | Redirected address, offset 68 |
| 8–9 | Displacement plus selected address | Peer | Selected address, offset 68 |

The closing read follows the halfword write and must reflect applicable overlap. Stage 10 has no closing pair; its displacement becomes the later **multiplier**.

These are represented ordering constraints, not guarantees of volatile access, atomicity, synchronization, or physical transaction counts.

### Successors, routes, and redirection

**Stages 1–2: successor discovery.** After displacement discovery, first-argument offset 84 receives the back-reference as a word. The successor is then read from selected-address offset 20, so it need not equal the carrier previously stored there. Successor offset 13 receives the character-converted selected address before closing.

Stage 2 additionally reads its redirected address from incoming-carrier offset 20 after that successor-byte write. Back-reference offset 84 then receives the first argument as a word before closing.

**Stages 3–5: character-address routing.** Each route is observed after its displacement.

- Stage 3 writes the character-converted back-reference to selected-address offset 29 before closing.
- Stage 4 writes a fresh word from first-argument offset 100 to displacement offset 84. Route offset 13 then receives the converted incoming carrier. Only afterward is the central reference retained from selected-address offset 100, before closing.
- Stage 5 has four ordered writes before closing: converted selected address to first-argument offset 1; converted incoming carrier to selected-address offset 13; central reference as a word to selected-address offset 68; converted selected address to route offset 13.

Stage 5’s closing carrier is still a fresh read. It cannot automatically be replaced with the central reference just written.

**Stages 6–7: redirected closure.** Stage 6 reads its redirected address from incoming-carrier offset 116, then reads its successor from current-base offset 20. Peer offset 13 receives the converted redirected address before closing.

Stage 7 retains its base and discovers no successor. After displacement discovery, incoming-carrier offset 21 receives the converted selected address, and selected-address offset 68 receives the incoming carrier as a word. Only then is the redirected address read from peer offset 116. Both intervening writes can affect that read through overlap.

**Stages 8–9: displacement-relative successors.** Their successor source is displacement offset 20, without adding the base.

Before stage 9 discovers its successor, three ordered writes follow displacement discovery: current base to incoming-carrier offset 84; the raw low byte of snapshot 9 to current-base offset 17; selected address to peer offset 68. That raw byte is also retained for stage 10. Stage 9’s earlier entry byte instead came from snapshot 8.

After successor discovery, both stages require these ordered effects:

- Peer offset 84 receives the peer as a word.
- Selected address plus incoming carrier receives the central reference as a word.
- Selected-address offset 20 receives the peer as a word.
- A fresh word read from central-reference offset 68 supplies the converted byte at selected-address offset 13.
- Successor offset 13 receives the converted peer.

Closing follows. The central-reference-relative reads in stages 8 and 9 are separate observations.

### Link replacement and products

After stage 10 obtains its signed multiplier, a write pair occurs twice: multiplier offset 84 receives the multiplier as a word, followed by the converted stage-10 base at selected-address offset 17. Both pairs belong to the represented behavior.

Traversal begins at the product of multiplier and stage-10 peer. Three dependent word reads, using offsets 68, 116, and 20 respectively, identify a **link location**. Link-location offset 20 supplies a retained **old-link address**. That same location is then overwritten with the stage-10 base shifted left one bit.

The following quantities have distinct provenance:

| Quantity | Required origin |
|---|---|
| Replacement-sensitive product | Multiplier times a fresh word from old-link offset 20, read after replacement |
| Traversal result | Word at replacement-sensitive-product offset 116 |
| Scaled product | Multiplier times replacement-sensitive product |
| Final product | Multiplier times scaled product |

The traversal result is not an operand of the scaled or final product. The old-link-relative read must remain after replacement because overlap can change its value.

Before traversal-result discovery, multiplier offset 84 receives the multiplier, then selected-address offset 17 receives the converted old-link address. Invocation-associated offset 264 receives the traversal result before scaled-product formation.

### Late observations and writes

After scaled-product formation, multiplier offset 84 again receives the multiplier. Selected-address offset 17 then receives a byte whose initialization is **not established**. No earlier routing value or later traversal-result byte establishes its payload.

Conditional on meaningful continuation, the subsequent contract is:

- A first word from scaled-product offset 68 is retained before retaining a word from first-argument offset 116.
- Selected-address offset 100 receives the traversal result, then selected-address offset 9 receives the character-converted first retained word.
- A second, fresh word from scaled-product offset 68 supplies the character-converted byte at scaled-product offset 17.
- Multiplier offset 84 again receives the multiplier.
- Twice-scaled-product offset 17 receives the raw low byte of the traversal result.
- The word at final-product offset 68 is then retained.
- The final destination is subsequently retained from first-argument offset 20.

The concluding byte writes have this order:

| Destination | Character-conversion input |
|---|---|
| Traversal-result offset 1 | Retained first-argument-offset-116 word |
| Final-destination offset 17 | Retained final-product-offset-68 word |
| Final-destination offset 9 | Twice the scaled product |

Both final-destination writes use the retained address, even if intervening overlap changes its original source. Both uses of twice the scaled product involve multiplication. The two scaled-product-offset-68 reads remain independent observations.

### Invocation-associated state and continuation

After the late addressed-memory effects, represented 32-bit assignments affect invocation-associated storage in this order:

| Group | Offsets and payloads |
|---|---|
| Shift results | 16: first argument shifted left four bits; 52: first argument shifted left eight bits |
| Clearing | Zero at 228, 96, 168, then 440 |
| Additional values | 36: four-bit-shift result; 76: eight-bit-shift result; 296: traversal result; 316: eight-bit-shift result; 380: traversal result; 392: eight-bit-shift result |

Offset 264 received the earlier traversal-result assignment, subject to intervening overlap. These offsets describe represented storage assignments, not a portable frame layout.

F-opaque-01 is invoked through the incoming target with the fifth argument’s storage address and two zeros. No explicit result is consumed.

If it returns, seventh-argument storage receives zero. Invocation-associated offset 412 then receives `0x08400e1c`, followed by zeros at offsets 256, 288, and 320. A separate external word is read afterward and passed to F-cdec09f5d432 with three zeros.

The represented continuation applies F-opaque-02 with operands zero and `0x08401154`. Its represented result supplies F-opaque-03’s invocation target; no explicit arguments are visible. Reaching or completing that continuation is unresolved.

## Side effects

Directly represented effects include mixed-width memory writes, overlap-sensitive address discovery, link replacement, argument-storage modification, invocation-associated assignments, storage-address exposure, and external control transfer.

No addressed regions are established as disjoint. This includes external sources, argument storage, invocation-associated storage, and derived destinations. Retaining an address preserves its numeric value, not the contents reachable through it.

No direct allocation, deallocation, ownership transfer, lifetime extension, rollback, synchronization, diagnostic channel, or recoverable-error protocol is established. Dependency behavior may add effects that are unavailable here. Passing the fifth argument’s storage address does not establish permission to retain it beyond its lifetime.

No dependency is known to preserve globals, argument storage, invocation-associated state, or reachable memory. Pre-call assignments do not guarantee post-call values.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-opaque-01 | Incoming-target invocation with a storage address and two zeros | Convention, implicit inputs, mutation, preservation, retention, ownership, release, failure, return |
| F-cdec09f5d432 | Invocation with a newly observed external word and three zeros | Implicit inputs, effects, preservation, ownership, release, failure, continuation |
| F-opaque-02 | Undefined-instruction operation with the stated operands | Handling, resumption, meaning of its represented result |
| F-opaque-03 | Invocation through that result without visible explicit arguments | Target validity, implicit inputs, effects, ownership, release, failure, termination |

These unresolved dependency contracts are **documented_only**. The IDs identify roles, not necessarily distinct runtime targets. F-opaque-02 is not established as an ordinary callable function.

The ten routing sources and the separate late source are external word observations, not established calls. Constancy, volatility, and concurrency semantics are unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They require valid reached arithmetic and accesses. Criteria after the unresolved byte also require meaningful continuation.

- **Initialization overlap:** If the initial halfword write changes donor offset 68, the first carrier must reflect the subsequent donor read.
- **Independent external observations:** If permitted memory semantics yield different values for a stage’s two source observations, selection uses the first and recipient offset 68 receives the second.
- **Successor replacement:** If an early back-reference write changes selected-address offset 20, successor discovery uses that changed word, not the previously delivered carrier.
- **Direct route address:** A route that promotes to 12 denotes address 12 directly. An additional pointer lookup would violate the established relationship.
- **Central-reference distinction:** If a first-argument snapshot differs from stage 4’s selected-address-offset-100 word, stage 5 must deliver the latter as the central reference.
- **Closing freshness:** A closing halfword overlapping the closing-carrier source can change that carrier, including after stage 5’s central-reference write.
- **Redirection timing:** Stage-7 writes overlapping peer offset 116 must be visible to its later redirected-address read under the applicable memory semantics.
- **Displacement-relative discovery:** Displacement 24 makes the stage-8 or stage-9 successor source address 44, not current base plus 44.
- **Raw-byte retention:** Snapshot 9 with low byte `0xD2` supplies byte `0xD2` to stage 10 independently of character signedness.
- **Replacement sensitivity:** Overlap between link replacement and old-link offset 20 requires the product to use the post-replacement word.
- **Product independence:** Multiplier 3 and replacement-sensitive product 60 imply scaled product 180 and final product 540, regardless of the traversal-result value.
- **Late freshness:** An intervening overlapping write can alter the second scaled-product-offset-68 observation without changing the first retained value.
- **Arithmetic validity:** Character narrowing does not make an otherwise invalid negative signed-left-shift operand valid.
- **Dependency mutation:** If F-opaque-01 changes the separate late source and returns, the later invocation receives the subsequently observed value.
- **Uninitialized payload:** Substituting zero, an earlier routing byte, or the traversal result’s low byte for the unresolved payload is unsupported.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and delivery of additional incoming state.
- External-source initialization, mutability, volatility, concurrency semantics, and physical access counts.
- Storage extent, overlap, alignment, permissions, lifetime, and access compatibility.
- Character signedness, out-of-range narrowing, byte order, and integer-to-address interpretation.
- Concrete argument-storage and invocation-associated layouts.
- Initialization and execution consequences of the unresolved byte.
- Dependency conventions, implicit state, mutation, preservation, retention, ownership, release, failure, and return.
- Undefined-instruction handling, final-target validity, and ordinary completion.

These uncertainties do not authorize invented payloads, assumed disjointness, wrapping arithmetic, memory-management guarantees, or guaranteed progress. No reporting mechanism for invalid arithmetic, invalid access, or the unresolved byte is established.

## Evidence

**Established static evidence:** The supplied representation supports the specified widths, address relationships, relative ordering, retained-value origins, product dependencies, storage assignments, and visible dependency operands.

**Inferred intent:** The effects resemble memory-state routing and link replacement. They do not establish a particular application operation, container type, or resource-management policy.

**Evidence limits:** Embedded comments, prior prose, and reviewer feedback are not independent behavioral evidence. Dependency implementations and complete execution semantics are unavailable. The unresolved byte prevents an unconditional end-to-end guarantee and may also preclude a guaranteed observable prefix.

**No runtime tests have been performed.**
