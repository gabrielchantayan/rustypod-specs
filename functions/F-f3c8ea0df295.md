<!-- Trusted provenance: F-f3c8ea0df295; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3c8ea0df295

## Purpose

F-f3c8ea0df295 traverses descriptor-directed relationships between an origin address, a target address, and a requested descriptor value.

Two independently observable results matter:

- The pointer-shaped return value.
- A caller-provided 32-bit status field, which the function may set to 1 but never explicitly clears.

A null return can accompany a successful status write. A nonnull return can occur without a status write. Callers therefore cannot interpret either result alone as a uniform success indicator.

The apparent intent is to search object relationships described by metadata, including relationships with indirectly obtained displacements. This interpretation is inferred. The established contract is the addressing, comparison, traversal, and signaling behavior below.

## Interface

### Declared inputs and result

The supplied function representation declares five parameters:

| Input | Established use |
|---|---|
| Origin address | Base for candidate-address calculation; also the location of an address-bearing word when indirect displacement is selected. |
| Target address | Compared with the origin or a calculated candidate address. |
| Descriptor address | Passed to classification dependencies and used as the base for descriptor-field reads. |
| Requested descriptor | A 32-bit value used in direct descriptor comparisons and as an explicit argument to a comparison dependency. |
| Status address | Address of a 32-bit integer that may be read after recursion and written with 1 on designated matching paths. |

The return is pointer-shaped in the recovered 32-bit address domain. Depending on the path, it is:

- Null.
- An address within a descriptor’s entry array.
- A result returned by a recursive invocation.
- At scan exhaustion, the most recently retained recursive result, or null if no entry recursion occurred.

### Anomalous recursive call

The declared five-parameter interface is not represented uniformly at every recursive call site.

First-form recursive forwarding supplies **eight arguments**, in this order:

1. Current origin.
2. Current target.
3. The 32-bit value read at descriptor offset +8.
4. Requested descriptor.
5. Status address.
6. Current invocation’s origin.
7. Current invocation’s target.
8. Current invocation’s descriptor.

The first five arguments describe the apparent recursive operation. The last three duplicate values from the current invocation rather than the forwarded descriptor value.

This eight-argument call representation is established static evidence. The semantic significance of the additional arguments, whether they are consumed, and the reliability of the recovered calling convention are **documented_only**. They must neither be omitted from the recovered interface account nor promoted into three additional declared parameters.

Entry-array recursion is represented with only the five declared arguments.

### Widths and layout

Addresses and ordinary integer fields are represented as 32-bit quantities. Some dependency results and intermediate carriers are represented as 64-bit quantities; the relevant classification decisions, comparison decisions, and returned pointer values use their low 32 bits.

Descriptor layout established by accesses:

| Byte offset | Meaning established by use |
|---|---|
| +8 | A 32-bit value used by the first descriptor form. |
| +12 | A 32-bit entry count used by the second descriptor form. |
| +16 | Start of the second form’s entry array. |

Each entry occupies eight bytes:

| Entry-relative offset | Meaning established by use |
|---|---|
| +0 | A 32-bit descriptor value. |
| +4 | A 32-bit packed displacement-and-flags word. |

Packed-word interpretation:

- Bit 0 selects indirect displacement.
- Bit 1 controls entry recursion and the status behavior of a local match.
- Bits 8–31 supply a signed displacement through arithmetic right shift by eight.
- Bits 2–7 have no explicit use in this function.

The extracted displacement ranges from −8,388,608 through 8,388,607. It is not an unsigned 24-bit displacement.

### Preconditions

No local validation establishes pointer validity, descriptor size, count validity, alignment, recursion depth, or safe address arithmetic.

Nonfaulting execution requires every location actually accessed on the selected path to be accessible, including locations accessed by dependencies. Their complete requirements are unavailable.

The function does not initialize the status. An initially nonzero status can affect control flow after entry recursion even if no invocation writes a new success value.

## Behavior

### Ordered classification and external reads

Classification uses two distinct external global locations as discriminator sources. Their contents are read at the corresponding calls; they are not established constants or snapshots taken at function entry.

The ordering is:

1. Invoke F-e3253cb77678 with the descriptor address.
2. Read the first external discriminator location and pass its current value, together with the preceding result, to F-061c2693b838.
3. If this classification returns nonzero, select the first descriptor form.
4. Only if it returns zero, invoke F-e3253cb77678 again with the descriptor address.
5. Read the second, separate external discriminator location and pass its current value, together with this new result, to F-061c2693b838.
6. If the second classification’s low 32 bits are zero, return null. Otherwise, select the second descriptor form.

The second discriminator is not read on a successful first classification path.

These reads occur after the corresponding F-e3253cb77678 invocation. Consequently, a dependency that changes external state can affect the discriminator value subsequently supplied to classification. The first classification call can also affect state observed during the second attempt. The function does not preserve entry-time discriminator values.

The two global locations’ identities and category meanings are unresolved.

### First descriptor form

If origin and target differ, the function immediately takes the first-form recursive forwarding path. It reads the field at descriptor offset +8 and makes the eight-argument recursive call described under Interface, returning the recursive result directly.

There is no local status check after this forwarding call.

If origin equals target, the function reads the 32-bit field at descriptor offset +8 and compares it with the requested descriptor.

- Equality causes a 32-bit status write of 1 and a null return.
- Inequality invokes F-f4ffd1965bfa twice: first with no explicit argument in the supplied representation, then with the requested descriptor as an explicit argument.
- Equality of those two 32-bit comparison results also causes a status write of 1 and a null return.
- Unequal results lead to first-form recursive forwarding.

The first F-f4ffd1965bfa invocation’s input is not established. It is therefore not justified to describe these calls as a proven semantic comparison between the descriptor field and requested descriptor.

On forwarding after failed comparison, the field at descriptor offset +8 is read again to obtain the recursive descriptor argument. A value previously read for comparison is not visibly retained for forwarding. Dependency mutations can therefore affect the forwarded value.

### Second descriptor form: scan bounds

The first entry address is descriptor address plus 16 bytes. Entries advance in eight-byte steps.

Before processing each entry, the function reads the count at descriptor offset +12 and forms an endpoint from the descriptor address, eight times that count, and the 16-byte header size. If the current entry address is at or beyond the endpoint under the recovered pointer comparison, scanning ends.

The count is reread at subsequent bounds checks. It is not captured once for the scan. Mutations performed by dependencies or recursive invocations can affect later scan bounds.

A stable zero count makes the initial entry address equal to the endpoint, so no entry is processed. Negative counts, overflowing count multiplication, endpoint wraparound, and the precise machine-level ordering relation for malformed addresses are not safely characterized as ordinary empty scans.

The retained recursive result initially has a zero low 32-bit value. At exhaustion, the function returns that retained result. Upper portions of the recovered intermediate carrier do not establish an additional pointer result.

### Candidate-address calculation

For each visited entry, the function first reads the packed word to test bit 0.

With bit 0 clear, it obtains the signed displacement from a subsequent packed-word read and adds that displacement to the origin to form the candidate address.

With bit 0 set, the visible order is:

1. Read the address-bearing 32-bit word at the origin.
2. Read the packed word again and extract its signed displacement.
3. Add that displacement to the address-bearing value.
4. Read a 32-bit integer from the resulting location.
5. Use that integer as the displacement added to the origin.

Thus, indirect mode does not simply add the encoded displacement to the origin. The encoded displacement locates a second displacement through an object-supplied address.

The bit-0 decision and displacement extraction are separate reads. The function does not establish an immutable packed-word snapshot, and the intervening origin read in indirect mode is observable in the supplied representation.

There are no local null, range, or alignment checks.

### Local entry matching

The calculated candidate address is compared with the target.

If they differ, the entry cannot return through the local-match path; it proceeds to the recursion decision.

If they are equal, the entry’s descriptor value is compared with the requested descriptor.

- Direct equality qualifies the entry.
- On inequality, F-f4ffd1965bfa is called twice in the same explicit-argument pattern described for the first form.
- Equal low 32-bit results qualify the entry.
- Unequal results send the entry to the recursion decision.

For a qualifying entry, bit 1 is read from the packed word:

| Bit 1 | Local result |
|---|---|
| Clear | Return the current entry address without an explicit status write. |
| Set | Write 1 to the status and return the current entry address. |

Neither qualifying case recurses into that matched entry.

### Entry recursion and retained results

An entry that does not produce a local return is tested for bit 1.

If clear, scanning advances without recursion.

If set, the function reads the entry descriptor and recursively calls F-f3c8ea0df295 with five represented arguments:

1. Calculated candidate address as the new origin.
2. Unchanged target.
3. Entry descriptor value.
4. Unchanged requested descriptor.
5. Unchanged status address.

After the call returns, its pointer-shaped result replaces the previously retained recursive result. The function then reads the status.

- Nonzero status causes an immediate return of that recursive result.
- Zero status allows scanning to continue.

Subsequent nonrecursive entries do not clear the retained result. A subsequent recursive invocation replaces it, including when that later invocation returns null.

Accordingly, a child may return a nonnull entry address without setting the status, the parent may continue scanning, and the parent may ultimately return that same address at exhaustion.

## Side effects

### Explicit writes and reads

The only explicit nonlocal write is a 32-bit write of 1 through the status address. It occurs on:

- A qualifying first-form match when origin equals target.
- A qualifying local second-form match with bit 1 set.

There is no explicit zero write and no explicit object or descriptor write.

Control-dependent reads include:

- The two separate external discriminator locations, in the classification order specified above.
- Descriptor fields at offsets +8 and +12.
- Entry descriptor and packed-word fields.
- The origin’s first word in indirect mode.
- The indirectly addressed displacement.
- Status immediately after entry recursion.

A preexisting nonzero status does not by itself stop scanning before recursion: the explicit status check is after an entry-recursive call.

Repeated field reads, especially those separated by dependencies, must not be collapsed into assumed stable snapshots. The source establishes the major sequencing described here, not a complete instruction-level memory-order contract.

### Ownership and lifetime

No allocation, release, reference-count operation, or ownership transfer is explicit.

A locally returned entry address refers into descriptor storage. Its usable lifetime depends on that storage remaining valid; this function provides no independent ownership guarantee.

Dependency effects are unknown. In particular, dependencies must not be assumed to preserve globals, leave pointed-to storage unchanged, or avoid allocation and release. Valid continued execution after a dependency requires any subsequently accessed storage to remain accessible.

### Errors and termination

There is no explicit error result separate from ordinary returns, no local exception conversion, no cycle detection, and no recursion-depth limit.

Malformed descriptors, inaccessible memory, cyclic relationships, and excessive recursion have no locally specified recovery. Their environmental failure behavior is **documented_only**.

## Dependencies

| Opaque function ID | Established role | Unresolved aspects |
|---|---|---|
| F-e3253cb77678 | Receives the descriptor address before each attempted classification. | Result meaning, memory requirements, global effects, and stability across calls. |
| F-061c2693b838 | Receives the preceding result and the current value read from one of two external discriminator locations. Its zero/nonzero result selects the path. | Category meanings, complete return convention, and side effects. |
| F-f4ffd1965bfa | Supplies comparison results after direct descriptor inequality; the second invocation explicitly receives the requested descriptor. | First invocation’s inputs, equivalence semantics, full calling convention, and side effects. |
| F-f3c8ea0df295 | Performs first-form forwarding and entry recursion. | First-form trailing-argument semantics and termination for arbitrary structures. |

The external discriminator operands are data read from separate global locations, not additional function dependencies. No dependency contract establishes that either location remains unchanged across calls.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless a case says otherwise, assume valid storage, stable relevant fields, nonoverflowing address calculations, and no dependency mutation affecting the example.

1. **Neither form recognized.**  
   The first classification returns zero, then the second classification’s low 32 bits are zero. The result is null with no explicit status write. Both normalization invocations and both ordered discriminator reads occur.

2. **First-form direct success.**  
   The first classification succeeds, origin equals target, and the field at descriptor offset +8 equals the requested descriptor. Status becomes 1 and the result is null. The second discriminator location is not read.

3. **First-form forwarding argument distinction.**  
   The first form is selected and origin differs from target. The recursive call’s third argument is read from descriptor offset +8. Arguments six through eight are the current origin, target, and descriptor, respectively. The parent returns the child result without a local status check. This example specifies the represented call, not the unknown significance of its trailing arguments.

4. **Discriminator changed before classification.**  
   The first F-e3253cb77678 invocation changes the first external discriminator location. The subsequent classification receives the changed value read after that invocation, not the value present at function entry.

5. **Second discriminator ordering.**  
   The first classification returns zero. If it or the second F-e3253cb77678 invocation changes the second discriminator location, the second classification receives that location’s value as read after the second normalization invocation. If the first classification instead succeeds, no second discriminator read occurs.

6. **Empty second-form scan.**  
   Only the second form is selected and its count remains zero. No entries are processed; the result is null with no explicit status write.

7. **Local entry match without bit 1.**  
   Direct displacement produces the target, the entry descriptor equals the requested descriptor, and bit 1 is clear. The function returns the entry address without writing status. An initially zero status remains zero under the stated no-mutation assumptions.

8. **Local entry match with bit 1.**  
   The same match with bit 1 set returns the entry address and writes status 1. No entry recursion occurs.

9. **Indirect displacement.**  
   Bit 0 is set and the encoded displacement is −8. The origin’s first word identifies an address-bearing base. The 32-bit value read eight bytes before that base is 24. The candidate is origin plus 24, not origin minus 8.

10. **Retained result and replacement.**  
    A child returns a nonnull value while status remains zero. If the remaining entries do not recurse or return locally, exhaustion returns that value. If a later child returns null while status remains zero, that null replaces the earlier result.

11. **Inherited nonzero status.**  
    Status is nonzero before entry recursion and remains so afterward. The parent immediately returns that child’s result, even if it is null and the child made no status write.

12. **Fallback comparison acceptance.**  
    Direct descriptor equality fails, but the two F-f4ffd1965bfa results have equal low 32 bits. The applicable local-match outcome follows. This does not establish what semantic equivalence the dependency implements.

## Uncertainties

The following remain **documented_only**:

- Semantic identities of the descriptor forms, requested descriptors, and external discriminators.
- Meaning and calling-convention reliability of the three additional first-form recursive arguments.
- Inputs to the first F-f4ffd1965bfa invocation and the meaning of its result.
- Complete dependency signatures, side effects, ownership effects, and storage-validity guarantees.
- Exact platform behavior for overflowing arithmetic, malformed extents, negative counts, and invalid address comparisons.
- Caller-supported aliasing between status, descriptor storage, and object storage.
- Concurrency guarantees and behavior under concurrent mutation.
- Environmental handling of invalid accesses and unbounded recursion.

The absence of explicit writes to descriptor storage does not establish immutability: dependencies or aliasing may affect it. Likewise, separate global discriminator locations do not imply constant contents.

## Evidence

This contract is based on static inspection of the supplied function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

Established evidence includes the five declared parameters; the anomalous eight-argument first-form recursive call; the five-argument entry-recursive call; separately ordered external discriminator reads; descriptor offsets and entry stride; signed displacement extraction; direct and indirect address formation; repeated bounds reads; path-specific status writes; and retained recursive return behavior.

Object-model terminology is inferred intent only. No dependency bodies, external-global definitions, complete caller contracts, or runtime traces were supplied. No runtime tests have been performed.
