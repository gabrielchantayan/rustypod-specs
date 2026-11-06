<!-- Trusted provenance: F-0ed683ddd57d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ed683ddd57d

## Purpose

F-0ed683ddd57d accepts an address-like 32-bit value, validates metadata immediately before that address, updates accounting words through a captured context address, and invokes dependencies with that context and the metadata address.

A zero input returns immediately. A nonzero input follows a validation and accounting path. Validation failure invokes a dependency identified as nonreturning, before this function performs any direct accounting writes.

The behavior is consistent with object-release bookkeeping, but that is **inferred intent**, not an established memory-management contract. The supplied evidence does not establish that any dependency frees, recycles, retains, or transfers ownership of storage.

## Interface

### Input and result

The single input is represented as a 32-bit signed integer and used to construct addresses. This contract calls it the **object address**.

- Zero requires no accessible metadata or context.
- A nonzero value leads to reads before the object address.
- There is no explicit return value or status result.
- Ordinary completion means that the dependencies on the selected path returned and this function returned. It does not independently prove successful memory reclamation.

The representation uses 32-bit integer words and unsigned 16-bit selectors. Address formulas below describe the supplied 32-bit representation; they do not establish portable behavior for overflowing signed address arithmetic.

### Metadata layout

The **metadata address** is eight bytes before the object address.

| Relative to object address | Width | Use |
|---|---:|---|
| −8 bytes | 32 bits | Validation word and subsequently reread accounting source |
| −4 bytes | 16 bits, unsigned | First accounting selector |
| −2 bytes | 16 bits, unsigned | Second accounting selector |

An accounting quantity is obtained by masking a metadata-word read with `0x3ffffffc`. The result retains bits 2 through 29 and clears bits 0, 1, 30, and 31. It ranges from zero through `0x3ffffffc`, in multiples of four.

Neither selector receives a visible bounds check. The accounting quantity receives no validation beyond masking.

### Context layout

After two preliminary calls, the function reads a global context reference and retains the resulting context address. All subsequent context-relative operations use that captured address.

| Context-relative byte offset | Update |
|---|---|
| `0x224 + 4 × first selector` | Subtract first accounting quantity |
| `0x30c` | Subtract first accounting quantity |
| `0xe0 + 4 × second selector` | Subtract first accounting quantity |
| `0x220` | Subtract first accounting quantity |
| `0x310 + 4 × index-producing dependency result` | Decrement by one |
| `0x390` | Decrement by one |
| `0x08` | Subtract freshly read accounting quantity |

Each accessed accounting word is 32 bits wide. These formulas identify effective addresses, not proven array boundaries, semantic field names, or mutually disjoint storage regions.

The unsigned 16-bit selectors range from 0 through 65,535. Their scaled values range from 0 through `0x3fffc`, so multiplying either selector by four does not itself overflow 32 bits. Adding the context address and offsets remains subject to the target’s address-arithmetic rules.

### Preconditions for ordinary completion

For a nonzero input, ordinary completion requires:

- Every metadata and selector read actually reached to address readable storage.
- Each accounting destination actually reached to support the required 32-bit read and write.
- The captured context to remain usable for the accesses and calls that use it.
- Validation to pass.
- Dependencies on the ordinary path to return normally.

No context-null check, selector-range check, dependency-result range check, alignment check, or insufficient-counter check is visible. No nonaliasing or exclusive-access guarantee is established.

## Behavior

### Zero-input path

A zero object address returns without dependency calls, metadata reads, context-reference reads, or direct accounting writes.

Repeated input tests do not establish additional input states: the supplied representation contains no assignment changing the input between those tests.

### Preliminary calls and context capture

For a nonzero input, the function calls, in order:

1. F-187ab9ba4eb6, without explicit arguments.
2. F-2fd8826557a4, without explicit arguments.

Their return values are unused. Only after both calls does the function read the global context reference.

Consequently, the captured context is the value observed after those calls, not necessarily the entry-time value. The function does not visibly reload that reference later. A later dependency can change the global reference without changing the already captured address used by this function; whether the captured storage remains valid is a separate, unresolved issue.

### Validation

The function first reads the 32-bit word at the metadata address.

- If that read is zero, validation fails without a second validation read.
- Otherwise, the function reads the word again and tests bit 0.
- If bit 0 of the second read is set, validation fails.
- Otherwise, validation passes.

Failure invokes F-a817d8d58c10, identified as nonreturning. Its termination or control-transfer mechanism is unknown.

The two reads must not be collapsed into a single cached-word condition when describing changing storage. A nonzero first read followed by a zero second read passes the visible validation. Conversely, a first read with bit 0 set does not by itself determine failure if the second read has that bit clear.

No direct accounting write precedes validation. The preliminary dependencies have already executed, so the failure path is not established to be globally side-effect-free.

### Context preparation

After validation passes, the function calls F-c0fc6d033caa with the captured context address.

The result is unused. No preservation guarantee is established for metadata, selectors, counters, or globals. Accounting reads occur after this call, and the function does not revalidate the metadata before using it for accounting.

### First four accounting updates

After F-c0fc6d033caa returns, the function:

1. Reads the unsigned 16-bit first selector.
2. Forms the first indexed accounting address.
3. Reads the metadata word and derives the first accounting quantity.
4. Reads and writes the first indexed word, subtracting that quantity.
5. Reads and writes the word at context offset `0x30c`, subtracting the same quantity.
6. Reads the unsigned 16-bit second selector and forms the second indexed address.
7. Reads and writes that indexed word, subtracting the same quantity.
8. Reads and writes the word at context offset `0x220`, subtracting the same quantity.

The quantity is retained across all four subtractions. A metadata mutation during these writes does not change that retained quantity.

The second selector is not read until after the first two writes. If either write aliases its storage, the selector read uses the then-current contents rather than an entry-time snapshot.

The unsigned quantity makes these subtraction expressions 32-bit unsigned arithmetic in the supplied representation. Results are reduced modulo \(2^{32}\); no saturation or underflow rejection appears. Subtracting four from zero therefore stores `0xfffffffc`.

### Index-producing call and count updates

Next, the function calls F-09dcb55e8e0a without explicit arguments. Its result is represented as a signed 32-bit integer and selects the word at context offset `0x310 + 4 × result`.

The function then:

1. Reads, decrements, and writes that indexed word.
2. Reads, decrements, and writes the word at context offset `0x390`.

The result is not checked for negativity or bounds. The signed multiplication and address additions are not checked for overflow.

Both count updates are represented as signed 32-bit additions of negative one. For values above the minimum signed value, the ordinary arithmetic result is one less. Behavior when decrementing the minimum signed value is **documented_only**: the supplied representation alone does not establish a portable wraparound rule.

F-09dcb55e8e0a executes between the first four accounting writes and these updates. Its effects are unknown; it may change storage that earlier updates wrote or that later operations read.

### Final quantity update

The function updates the word at context offset `0x08` by subtracting a freshly derived accounting quantity.

This quantity comes from another metadata-word read masked with `0x3ffffffc`. It is not the quantity retained for the first four subtractions. It may differ because of an intervening dependency, an aliasing write, or an external mutation.

This subtraction again has unsigned 32-bit arithmetic behavior. The supplied expression does not establish the relative order of its counter-value read and metadata-value read; both supply the final subtraction before its result is stored.

### Final calls and return

After all seven direct accounting updates, the function calls:

1. F-cc6c9b4a0f68 with the captured context address and the metadata address.
2. F-e586bff64fb7 with the captured context address.

The second call occurs only if the first returns. If both return, this function returns.

No explicit rollback or error-result handling is present. Earlier direct writes remain performed if a subsequent dependency fails to return; the dependencies’ own effects are unknown.

## Side effects

### Direct memory effects

On the ordinary nonzero path, the function performs seven ordered 32-bit read-modify-write updates. Seven updates need not target seven distinct words.

There are **no separately identified stores to metadata or object contents**. This does not establish that those regions remain unchanged: an accounting destination can alias metadata or object storage, and a direct accounting write can therefore modify that storage.

Direct reads include:

- One global context-reference read.
- One or two validation-word reads.
- Two unsigned 16-bit selector reads.
- One metadata read supplying the first four subtractions.
- One fresh metadata read supplying the final subtraction.
- The accounting-value reads associated with the seven updates.

These are accesses identified by the supplied representation, not a claim about compiler-generated instruction counts, atomicity, or volatile-memory behavior.

### Aliasing and intervening mutation

No disjointness is established among accounting destinations, metadata, selectors, or object storage.

A later read uses the then-current contents of its address. When accounting destinations coincide, the later operation observes the earlier write’s result **only if nothing intervening changes that storage**.

This qualification matters across F-09dcb55e8e0a. That dependency runs after the first four updates and before the final three; its unknown effects can replace values written earlier. Even when two updates have identical destinations, the later update is not guaranteed to start from the earlier update’s stored result across that call.

Within the first four updates, no dependency call intervenes. Under sequential execution without external mutation, an overlapping later access therefore sees the result of earlier writes. Writes that overlap selectors or metadata can also affect later reads of those fields.

The operation is not visibly transactional, and no complete metadata snapshot is taken.

### Ownership and synchronization

F-cc6c9b4a0f68 receives the metadata address, not the object address. This establishes its argument, not ownership transfer or storage release.

The evidence does not establish that either address becomes invalid, that either address is retained, or that repeated processing is legal.

The positions of F-c0fc6d033caa and F-e586bff64fb7 suggest paired context operations, but locking, exclusion, and release semantics remain unknown. No atomicity or concurrency guarantee follows from their placement alone.

## Dependencies

| Opaque function ID | Explicit arguments | Established use |
|---|---|---|
| F-187ab9ba4eb6 | None | First nonzero-path call |
| F-2fd8826557a4 | None | Second call, before context capture |
| F-a817d8d58c10 | None | Nonreturning validation-failure destination |
| F-c0fc6d033caa | Captured context address | Runs after validation and before accounting |
| F-09dcb55e8e0a | None | Supplies the signed index for one count update |
| F-cc6c9b4a0f68 | Captured context address; metadata address | Runs after all direct accounting updates |
| F-e586bff64fb7 | Captured context address | Final call on the ordinary path |

No dependency is assumed to preserve globals or addressed storage, maintain context lifetime, acquire or release ownership, or provide synchronization.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where failure is specified, they assume valid reached addresses and normally returning dependencies.

### Zero input

For an object address of zero, the function returns without dependency calls or the identified memory accesses.

### Validation failures

If the first validation read is zero, F-a817d8d58c10 is called without a second validation read.

If the first read is nonzero and the second read has bit 0 set, the same dependency is called.

In either case, both preliminary calls and context capture have already occurred, but no direct accounting update has occurred.

### Validation reads disagree

If the first validation read is nonzero and the second is zero, validation passes. Accounting subsequently uses another metadata read after F-c0fc6d033caa; neither validation value fixes the accounting quantity.

### Zero accounting quantity

With stable metadata `0x00000002`, validation passes and the masked quantity is zero.

Assuming nonoverlapping storage and no dependency mutation:

- The four initial quantity updates store unchanged numeric values.
- Both count decrements occur.
- The final quantity update also stores an unchanged value.
- Both final dependencies are called.

A zero quantity does not select a no-op path.

### Distinct ordinary destinations

With stable metadata `0x00000040`, seven distinct destinations, no relevant dependency mutations, and count values away from signed overflow:

- Each of the five quantity updates subtracts 64.
- Each count update subtracts one.
- F-cc6c9b4a0f68 receives the metadata address as its second argument.

### Overlap before the index-producing call

A first selector of 58 places the first indexed destination at offset `0x30c`, also used by the second update.

If its initial value is 1000, the retained quantity is 64, and nothing else mutates the word, the successive stores are 936 and 872. Combining these into one subtraction would change behavior.

### Overlap across the index-producing call

Suppose the first indexed destination is at offset `0x390`, its initial value is 1000, and the first quantity is 64. The first update stores 936.

If F-09dcb55e8e0a subsequently replaces that word with 200, and no intervening indexed decrement targets it, the later fixed count decrement stores 199—not 935. The dependency’s mutation prevents the earlier stored value from determining the later result.

### Direct write aliases metadata

Suppose the first indexed destination is exactly the metadata word, that word is `0x00000040` when the first quantity is captured, and no relevant mutation intervenes.

The first subtraction stores zero into the metadata itself. The remaining initial quantity updates still use the retained quantity 64. If the metadata remains zero until the final accounting read, that final quantity is zero. There is no revalidation rejecting the changed metadata.

### Unsigned wraparound

If a quantity destination contains zero and the applicable quantity is four, the subtraction stores `0xfffffffc`. This example does not establish wraparound semantics for the separate signed count decrements.

## Uncertainties

The following remain **documented_only**:

- Dependency implementations, mutations, failure modes, and ownership effects.
- Context meaning, lifetime, and valid storage extent.
- Whether the metadata is an allocation header.
- Accounting units and intended counter categories.
- Intended selector bounds and valid dependency-result range.
- Endianness, alignment requirements, and invalid-address behavior.
- Exact target consequences of signed arithmetic overflow in address calculations and count decrements.
- Caller guarantees about aliasing, synchronization, and stable storage.
- Whether the failure dependency terminates, traps, or transfers control elsewhere.
- Whether processing an object more than once is supported.

These gaps do not justify assuming allocator, lock, lifetime, or error semantics.

## Evidence

The contract derives from the supplied static function representation and opaque dependency mapping.

Established static evidence includes the zero-input return, dependency order, context-capture timing, metadata widths and offsets, short-circuit validation reads, accounting mask, seven ordered updates, intervening index-producing call, final metadata reread, and final call arguments.

Aliasing consequences are conditional deductions from those accesses; disjointness is not established. Allocation release, statistical meanings, and paired synchronization are inferred interpretations rather than guarantees.

No runtime tests, dependency-body inspection, or dynamic observations have been performed.
