<!-- Trusted provenance: F-fce4478c64cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fce4478c64cd

## Purpose

F-fce4478c64cd selects a fixed data address from an unsigned selector. The selection depends on the complete selector value, including a two-bit mode field at mask `0xC0`.

The established behavior is address selection, not data retrieval: the function returns a pointer without reading the pointed-to contents. The supplied evidence does not establish what those contents mean, their type, their extent, or whether callers may modify them.

Two selector families have explicit results:

- Ordinary selectors from `0x00` through `0x1E`.
- Ten alternate selectors whose mode field is `0x80`.

All other selectors return one shared fallback address. Several explicitly supported selectors also share addresses; those aliases are part of the observable behavior.

## Interface

### Input

One unsigned integer selector, modeled here as a 32-bit bit pattern.

The relevant fields are:

| Field | Mask | Meaning established by the function |
|---|---:|---|
| Mode field | `0x000000C0` | Chooses ordinary selection, alternate selection, or immediate fallback |
| Alternate flag | `0x00000080` | Cleared when the mode field is exactly `0x80` |
| Remaining bits | `0xFFFFFF3F` | Not generally discarded; they remain relevant to exact-match selection |

The input is passed by value. There is no input pointer, buffer, length, or caller-provided output location.

The supplied representation uses an unsigned integer parameter and 32-bit masks. The original type definitions and calling convention are not supplied, so the exact ABI-level parameter width is **documented_only** beyond this 32-bit behavioral model.

### Output

The function returns one pointer to a fixed address.

For this contract, addresses are identified by opaque result labels:

- `O00` through `O1E` identify ordinary results where distinct labels are needed.
- `A01`, `A0A`, `A0B`, `A0C`, `A12`, `A15`, `A17`, `A18`, and `A19` identify alternate-only results.
- `D` identifies the shared fallback result.

These labels denote pointer identities, not decoded strings, numeric values, or allocated object types. An alias in the tables means exact pointer equality.

All return expressions in the supplied evidence designate nonzero fixed addresses. There is no explicit null-return path.

### Preconditions and ownership

No input-value precondition is needed for selecting a result: every 32-bit selector has a defined return path in the supplied function.

Successful use of the returned pointer requires additional environmental assumptions not established here:

- The corresponding address must be mapped if the caller dereferences it.
- The caller must know the object’s actual type, extent, alignment requirements, and access permissions.
- The caller must not infer ownership transfer or permission to release storage from this function.

The function itself neither allocates nor releases memory. Whether some external component owns, changes, or releases the addressed storage is unknown.

## Behavior

### Selection rules

The mode field is examined before the ordinary selector mapping is used.

**Mode field `0x00`: ordinary selection.**  
The complete input must match one of the ordinary values in the table below. The function does not reduce the input to its low six bits. An input with other high bits set therefore falls back unless its full value is explicitly supported.

**Mode field `0x80`: alternate selection.**  
Only bit `0x80` is cleared, using the 32-bit mask `0xFFFFFF7F`. The resulting complete value is matched against the alternate mapping. No other high bits are cleared.

**Mode field `0x40` or `0xC0`: fallback.**  
Every input in either mode returns `D`, regardless of its remaining bits.

An ordinary or alternate lookup that has no explicit match also returns `D`.

This is not a wrapping lookup or a general “base selector plus optional flag” interface. In particular, the alternate flag is accepted only for the ten listed alternate selectors.

### Ordinary mapping

The following table applies only when the complete input equals the listed value.

| Input | Returned identity |
|---:|---|
| `0x00` | `O00` |
| `0x01` | `O01` |
| `0x02` | `O02` |
| `0x03` | `O03` |
| `0x04` | `O04` |
| `0x05` | `O05` |
| `0x06` | `O06` |
| `0x07` | `O07` |
| `0x08` | `O08` |
| `0x09` | `O09` |
| `0x0A` | `O0A` |
| `0x0B` | `O0B` |
| `0x0C` | `O0C` |
| `0x0D` | `O0D` |
| `0x0E` | `O0E` |
| `0x0F` | `O0F` |
| `0x10` | `O10` |
| `0x11` | `O11` |
| `0x12` | `O12` |
| `0x13` | `O13` |
| `0x14` | `O14` |
| `0x15` | `O05` |
| `0x16` | `O08` |
| `0x17` | `O17` |
| `0x18` | `O18` |
| `0x19` | `O19` |
| `0x1A` | `O1A` |
| `0x1B` | `O1B` |
| `0x1C` | `O1C` |
| `0x1D` | `O1D` |
| `0x1E` | `O1E` |

There are 31 supported ordinary inputs and 29 distinct ordinary addresses. The two ordinary aliases are:

- `0x05` and `0x15`.
- `0x08` and `0x16`.

No other ordinary result addresses coincide in the supplied evidence.

### Alternate mapping

The table lists original inputs, not just their values after clearing the alternate flag.

| Original input | Value after clearing `0x80` | Returned identity |
|---:|---:|---|
| `0x80` | `0x00` | `O00` |
| `0x81` | `0x01` | `A01` |
| `0x8A` | `0x0A` | `A0A` |
| `0x8B` | `0x0B` | `A0B` |
| `0x8C` | `0x0C` | `A0C` |
| `0x92` | `0x12` | `A12` |
| `0x95` | `0x15` | `A15` |
| `0x97` | `0x17` | `A17` |
| `0x98` | `0x18` | `A18` |
| `0x99` | `0x19` | `A19` |

The alternate-zero result is exactly the ordinary-zero address. Each of the other nine alternate results is distinct from every ordinary result, from the other alternate results, and from `D`.

For example, ordinary inputs `0x05` and `0x15` alias, but this does not create an alternate alias: `0x85` returns `D`, whereas `0x95` returns `A15`.

### Address layout

The supplied fixed addresses establish the following relative byte offsets. The address identified by `A12` is the reference origin. These offsets describe address relationships only; they do not establish a single allocated array or the sizes of individual objects.

| Result identity or sequence | Relative byte offset |
|---|---:|
| `A12` | `0x00` |
| `A0A` | `0x04` |
| `O00` through `O07`, in numeric order | `0x08` through `0x24`, in increments of four |
| `O08` | `0x28` |
| `O09` through `O14`, in numeric order | `0x2C` through `0x58`, in increments of four |
| `O17` through `O1E`, in numeric order | `0x5C` through `0x78`, in increments of four |
| `A01` | `0x7C` |
| `A0B` | `0x80` |
| `A0C` | `0x84` |
| `A15` | `0x88` |
| `A17` | `0x8C` |
| `A18` | `0x90` |
| `A19` | `0x94` |
| `D` | `0x98` |

Selector numbers in the sequences are hexadecimal. No separate `O15` or `O16` identity exists because those selectors return `O05` and `O08`.

The four-byte spacing must not be interpreted as proof of four-byte strings, four-byte records, or pointer-sized entries. No contents are supplied.

### Integer boundaries and comparisons

The operation that clears the alternate flag is bitwise masking, not subtraction or arithmetic normalization. It cannot carry, borrow, or overflow.

The supplied control flow includes signed comparisons after clearing the flag. Under the 32-bit model, values with bit 31 set may enter a different comparison branch from small positive values. They still cannot satisfy any of the explicit supported-value equalities and therefore return `D`.

Consequently, the mapping above remains complete even for inputs near the unsigned and signed boundaries. Neither sign interpretation nor high-bit retention introduces an additional successful selector.

## Side effects

The established function-local behavior has no externally visible writes other than delivering its return value.

Specifically, the supplied body:

- Does not dereference an input pointer.
- Does not read the contents at any selected return address.
- Does not write global storage or returned storage.
- Does not allocate or release memory.
- Does not invoke another function.
- Does not maintain a counter, cache, or other persistent state.
- Does not expose an error status separately from the returned pointer.

There is therefore no ordering contract for data-memory reads or writes within this function: none are shown. Selection evaluates the mode before using the applicable mapping, but intermediate control-flow decisions produce no separate observable data effects.

Repeated calls with the same selector designate the same fixed address in the supplied program image. This does not guarantee that the bytes stored there remain unchanged between calls.

Unsupported input is represented by `D`, not by a demonstrated exception, trap, null pointer, or diagnostic. Environmental failures outside the shown body are not specified.

## Dependencies

No function dependencies are present in the supplied body.

F-fce4478c64cd depends on fixed address destinations, whose contents and storage definitions were not supplied. No opaque function IDs for storage initialization, decoding, allocation, or release can be established from this evidence.

No unknown callee behavior is assumed, because there are no calls in the shown function.

## Acceptance examples

These are static acceptance criteria, not reports of executed tests.

| Condition | Required result |
|---|---|
| Input is `0x00` | Return `O00`. |
| Input is `0x80` | Return the exact same pointer as input `0x00`. |
| Inputs are `0x05` and `0x15` | Return equal pointers, both `O05`. |
| Inputs are `0x08` and `0x16` | Return equal pointers, both `O08`. |
| Input is `0x01` versus `0x81` | Return different pointers: `O01` versus `A01`. |
| Input is `0x0A` versus `0x8A` | Return different pointers: `O0A` versus `A0A`. |
| Input is `0x92` | Return `A12`, not `O12`. |
| Input is `0x85` | Return `D`, despite ordinary selector `0x05` being supported. |
| Input is `0x96` | Return `D`, despite ordinary selector `0x16` being supported. |
| Input is `0x1E` | Return `O1E`, the final supported ordinary result. |
| Input is `0x1F` or `0x20` | Return `D`. |
| Input is `0x40` or `0xC0` | Return `D`; the mode alone excludes both mappings. |
| Input is `0x100` | Return `D`, not `O00`; high bits are not discarded. |
| Input is `0x180` | Return `D`; clearing `0x80` leaves `0x100`. |
| Input is `0x80000080` | Return `D`; signed comparison does not make it a supported alternate selector. |
| Input is `0xFFFFFFFF` | Return `D` because its mode field is `0xC0`. |

If the addressed storage is valid and inspected by an external observer, selection alone must not change its bytes. No acceptance criterion can be stated for the bytes’ values without additional evidence.

## Uncertainties

### Inferred intent

The two mode families and small fixed-address destinations suggest a selector-to-descriptor or selector-to-label lookup. That interpretation is **inferred intent**, not an established semantic type.

Likewise, `D` behaves as a fallback address, but its contents might represent an error label, an empty value, or something else. No meaning is assigned to it here.

### Documented-only unknowns

The following remain **documented_only**:

- Original ABI parameter and return-pointer widths.
- Original integer type definitions.
- Data contents, encoding, terminators, and object boundaries.
- Readability, writability, lifetime, and ownership of the returned storage.
- Whether the nearby addresses belong to one storage object or multiple objects.
- Whether other components mutate or invalidate the selected storage.
- Any higher-level meaning of the mode bits or supported selectors.

These unknowns do not prevent specifying pointer selection and aliasing. They do prevent treating the result as a validated string, record, or owned allocation.

## Evidence

The supplied static function body establishes the mode mask, alternate-flag clearing, explicit selector matches, shared fallback, ordinary aliases, alternate-zero alias, and fixed relative address layout.

The 32-bit behavioral model follows the supplied unsigned parameter representation and masks; the original ABI declarations were not provided. Signed-comparison consequences above are stated within that model.

There are 41 explicitly supported input values mapping to 38 distinct non-fallback addresses, plus one distinct fallback address. These counts follow from the complete mappings, not from runtime observation.

No runtime tests, memory inspections, or dependency implementations have been provided or exercised.
