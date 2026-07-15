# Serialization

Singulisp provides built-in language support for converting values to byte sequences (serialization) and restoring values from byte sequences (deserialization). Choose between the following two formats according to the use case.

| Format | API | Schema | Primary uses |
|------|-----|----------|----------|
| Schema-evolution format | `serialize` / `deserialize` / `try-deserialize` | Includes a type descriptor in the header and can migrate field additions, removals, and reorderings | Save data and long-term storage |
| Raw BBin | `bbin-write` / `bbin-read` / `schema-id` | Has no header or type descriptor and requires an exact schema match | Communication, continuous streams, and state hashes |

**BBin v1** is the positional body encoding shared by both formats. The schema-evolution format adds a fixed header and type descriptor to that body, while raw BBin reads and writes only the body.

`persist`, `thaw`, and `freeze` are separate region-related operations independent of BBin serialization. See [Regions](regions.md).

---

## Schema-evolution Format

### Serialization

```lisp
(serialize (& value))      ;; -> [Vec u8]
;; Or the type-qualified form
(T/serialize (& value))    ;; -> [Vec u8]
```

Converts a value to a byte sequence in BBin v1 format. The argument is a shared reference and the return value is `[Vec u8]`.

### Deserialization

```lisp
(T/deserialize (& bytes))         ;; -> [Result T String]
(deserialize (& bytes) : T)       ;; type-annotation form (when context does not uniquely determine the type)
```

Restores a value of type `T` from a byte sequence. It returns `(Ok value)` on success or `(Err message)` on failure. The error type is `String`, containing a human-readable description. If the schema has changed, the migration path runs.

### Checked Deserialization (`try-deserialize`)

```lisp
(T/try-deserialize (& bytes))     ;; -> [Result T DeserError]
(try-deserialize (& bytes) : T)   ;; type-annotation form
```

This checked entry point returns a `DeserError` for malformed input without aborting the process. `try-deserialize` checks buffer bounds, enum tags, collection lengths, and recursion depth. When schema IDs differ, it validates the writer descriptor and body before migration. `deserialize` is a schema-aware entry point that performs schema migration, but its body decoding uses an unchecked fast path; use `try-deserialize` for untrusted byte sequences.

## Raw BBin for Identical Schemas

Raw BBin omits the fixed header and type descriptor and appends a value's body directly to the end of an existing buffer. It assumes that reader and writer use the same type definition and performs no migration.

```lisp
(let mut bytes (Vec/new))
(Packet/bbin-write (& packet) (& mut bytes))
;; The form that infers the type from the argument is equivalent
(bbin-write (& packet) (& mut bytes))

(let mut pos 0)
(match (Packet/bbin-read (& bytes) (& mut pos))
  [(Ok value) (handle-packet value)]
  [(Err error) (handle-invalid-packet error)])

;; Form specifying the type to read with an annotation
(bbin-read (& bytes) (& mut pos) : Packet)
```

`T/bbin-write` creates no return value and appends to `[Vec u8]`. `T/bbin-read` checks bounds, enum tags, collection lengths, and recursion depth, and returns `[Result T DeserError]`. Only on success does it advance `pos` by the number of bytes consumed; on failure, it preserves the position from before the call. This allows multiple values written consecutively to the same buffer to be read in order.

```lisp
(Packet/schema-id)  ;; -> u64
```

`T/schema-id` is an ID representing the type structure. In communication, compare the IDs on both sides during connection setup or a similar phase and exchange raw BBin only when they match. Because raw BBin itself contains no schema ID, calling `bbin-read` with a different type is not guaranteed to detect the mismatch.

---

## The `DeserError` Type

`DeserError` is a built-in enum predefined by the compiler; users do not define it.

| Variant | Payload | Description |
|-----------|-----------|------|
| `Truncated` | `i32` | The byte sequence ends prematurely |
| `BadTag` | `i32` | An enum tag is out of range |
| `BadSchema` | none | Invalid magic or version, or an inconsistent header size for an identical schema |
| `CycleDetected` | none | The recursion-depth limit (256) was exceeded, or an invalid migration descriptor was detected |
| `TooLarge` | `i32` | A length field exceeds the permitted limit |

An `i32` payload is auxiliary information for locating the error and does not necessarily represent a byte count or received tag.

---

## Automatic Generation of Serialization Functions

The compiler scans the program's CST and automatically generates and injects the following
functions for every serializable struct and enum only when it detects a reference to `serialize`,
`deserialize`, `try-deserialize`, `bbin-write`, `bbin-read`, or `*/schema-id`. If there is no such
reference, it generates nothing.

Generated functions for a type named `T`:

| Function | Signature | Description |
|--------|-----------|------|
| `T/serialize` | `[& T] -> [Vec u8]` | Serialization with a header |
| `T/deserialize` | `[& [Vec u8]] -> [Result T String]` | Deserialization with migration |
| `T/try-deserialize` | `[& [Vec u8]] -> [Result T DeserError]` | Checked deserialization |
| `T/bbin-write` | `[& T] [& mut [Vec u8]] -> ()` | Appends a headerless body to an existing buffer |
| `T/bbin-read` | `[& [Vec u8]] [& mut i32] -> [Result T DeserError]` | Reads an identical-schema body from the current position with validation |
| `T/schema-id` | `() -> u64` | Schema ID of the type-structure descriptor |

---

## Serializable Types

By **fixed-point iteration**, the compiler selects only types all of whose fields are serializable.

### Primitive Types (Always Serializable)

| Type | Encoding | Bytes |
|----|-----------|---------|
| `i8` | Two's complement | 1 |
| `i16` | Little-endian | 2 |
| `i32` | Little-endian | 4 |
| `i64` | Little-endian | 8 |
| `u8` | Unchanged | 1 |
| `u16` | Little-endian | 2 |
| `u32` | Little-endian | 4 |
| `u64` | Little-endian | 8 |
| `f32` | Normalized IEEE 754, LE | 4 |
| `f64` | Normalized IEEE 754, LE | 8 |
| `bool` | `0x00`(false) / `0x01`(true) | 1 |

### Composite Types

| Type | Encoding |
|----|-----------|
| `String` | `u32 byte_len` (LE) followed by raw bytes |
| `[Vec T]` | `u32 len` followed by elements, each recursively encoded |
| `[PVec T]` | Same as `[Vec T]`; `T` is limited to POD scalars (integers, `bool`, and floating-point types) |
| `[Array T N]` | Encoded directly as a fixed-length collection |
| `[SoaArray T N]` | Encoded directly as a fixed-length SoA collection |
| `[Option T]` | `u8` tag (0=None, 1=Some), followed by contents for Some |
| `[Box T]` | Same as the inner type; limited to `Box` with the global allocator, with no arena provenance |
| struct | Fields concatenated in declaration order (positional) |
| enum (`deftype`) | `u32 tag` (declaration-order index), followed by payload |
| `[Result T E]` | `u32 tag` (Ok=0/Err=1), followed by payload (`T` or `E`) |
| `[PMap K V]` | `u32 len`, followed by entries consecutively in positional `(K, V)` form |

Direct BBin support for `[PMap K V]` limits keys to integers and `bool`, and values to integers, `bool`, and floating-point types. Project any other key or value explicitly to a Wire type.

### Mathematical Types (Built-in Domain Types)

Mathematical types are encoded in storage-vector order, with each component encoded as `f32`.

| Type | Encoding | Bytes |
|----|-----------|---------|
| `Vec2` | x, y | 8 |
| `Vec3`, `Float3` | x, y, z | 12 |
| `Vec4`, `Quat` | x, y, z, w | 16 |
| `Mat3` | 3 columns × (x,y,z) | 36 |
| `Mat3r` | 3 rows × (x,y,z) | 36 |
| `Mat4` | 4 columns × (x,y,z,w) | 64 |
| `Mat4r` | 4 rows × (x,y,z,w) | 64 |
| `Mat3x4` | 3 rows × (x,y,z,w) | 48 |

### Non-serializable Types

Any struct or enum containing one of the following is excluded from serialization and produces a compile-time error:

- `GpuBuffer`—GPU device memory
- low-level SIMD types—`f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, `u32x4`
- `char`, references, and `StrSlice`
- arena and dynamic SoA collections (`SoaVec`, `SoaPVec`, `SoaPMap`)
- closure and `Fn` types
- `Reader` and `Writer`—OS file handles
- `ScopeHandle`, `ChannelHandle`, and `ModuleHandle`—host-side resources
- a struct with `#[serialize-opaque]` but no corresponding Wire conversion
- an unconcretized generic template; a struct or enum instance whose type arguments are determined at the use site is monomorphized and can be serialized when all its fields have supported types

---

## `#[serialize-opaque]` and Wire Conversion

```lisp
#[serialize-opaque]
(defstruct Celsius (kelvin : f64))

(defn Celsius/to-wire [(c : [& Celsius])] -> f64
  (- c.kelvin 273.15))

(defn Celsius/from-wire [(degrees : f64)] -> Celsius
  (Celsius (+ degrees 273.15)))
```

`#[serialize-opaque]` suppresses structural encoding of physical fields. When `T/to-wire` and `T/from-wire` are both present, the compiler uses the Wire type's structure for the descriptor and body and includes type `T` in BBin. Without the pair, the type is not serializable.

The standard library exposes the same contract as the `std::serialize::SerializeAs` trait. `HashMap`, `Heap`, `Deque`, and similar types are converted to and stored in a public Wire representation rather than as their physical tables or buffers. The Wire type itself must also be serializable.

---

## Binary Format (BBin v1)

The byte sequence output by the schema-evolution form `T/serialize` has the following three-part structure:

```
offset  0 : magic       (4 bytes)    "SGSV" = 0x53,0x47,0x53,0x56
offset  4 : format_ver  (2 bytes LE) = 1
offset  6 : schema_id   (8 bytes LE) FNV-1a 64-bit hash of the type structure
offset 14 : header_size (4 bytes LE) body start offset = 18 + descriptor length
offset 18 : descriptor  (variable)   recursive schema descriptor (for migration)
offset 18+descLen : body (variable)  positional BBin v1 data in declaration order
```

Because `header_size` is a compile-time constant—the descriptor length is statically known—serialization can write directly to the output buffer without an intermediate buffer.

Raw BBin's `T/bbin-write` outputs only the same bytes as the body. It does not add magic, format version, schema ID, header size, or a type descriptor.

### Floating-point Normalization

Floating-point values are normalized during serialization:

- `-0.0` → `+0.0` (sign bit cleared)
- `NaN` → canonical NaN (`f32`: 0x7FC00000, `f64`: 0x7FF8000000000000)

---

## Reading the Schema-evolution Format

### Fast Path (Matching Schema)

When magic, format_version, and schema_id all match, the body is decoded quickly in positional form from `header_size`, without descriptor parsing or string matching.

### Migration Path (Schema Mismatch)

When schema_id differs, the source descriptor embedded in the binary is matched against the target schema, converted to a positional body for the target schema, and then decoded.

Struct fields are matched by name. A target field absent from the source descriptor is filled with a default value:

| Type | Default |
|----|-----------|
| Integer type | `0` |
| Floating-point type | `0.0` |
| `bool` | `false` |
| `String` | Empty string `""` |
| vector / `Quat` / `Float3` | All components `0` |
| matrix type | Identity matrix |
| `[Vec T]` | Empty list |
| `[Option T]` | `None` |
| struct | Each field recursively default-constructed |

Field additions, removals, and reorderings are supported, as is recursive resolution of fields with the same Wire type. If the Wire kind changes even for a field with the same name—for example, to a different primitive, mathematical, or matrix ID—no numeric conversion occurs; the source value is skipped and the target field's default is used. If magic or format version does not match, `Err("serialize: header/schema mismatch")` is returned and migration is not attempted. A schema-ID mismatch is not itself an error; it selects the migration path.

Enum variants are matched by name, and payloads are resolved by declaration position within the variant. If a variant from the stored value is absent on the reading side, the reader default-constructs the first variant of the target enum.

### Recursive-type Support

Mutually referential types, such as a recursive type using `[Box Tree]`, are represented in finite length by encoding the descriptor with a **REF type-table scheme** (type-index references), allowing migration to operate correctly.

---

## Byte-identity Guarantee

If canonical bytes produced by the serializer are deserialized with the same schema and the resulting value is serialized again, the output is exactly identical to the original canonical bytes (the **byte-identity guarantee**).

---

## State Hash (FNV-1a 64-bit)

A state hash is the FNV-1a 64-bit value of a canonical BBin v1 byte sequence. It is not a compiler built-in. Compute it by creating a headerless body byte sequence with `T/bbin-write` or a similar function and applying `hash/bbin-fnv1a` from `std::hash`.

```
offset basis: 0xCBF29CE484222325
prime:        0x100000001B3
for each byte b: h = (h XOR b) * prime (mod 2^64)
```

As long as the same schema and value produce the same body byte sequence, their state hashes also match. The `Hash` trait used for `HashMap` and `HashSet` keys is an internal, implementation-defined hash whose only contract is consistency with Eq; it is independent of the state hash. Do not use `Hash/hash` for storage, communication, or checksums.

---

## Examples

### Basic Save and Load

```lisp
(defstruct Player
  (name  : String)
  (level : i32)
  (hp    : f32))

;; Save
(defn save-player [(p : Player)] -> [Vec u8]
  (serialize (& p)))

;; Load (errors are String messages)
(defn load-player [(bytes : [Vec u8])] -> [Result Player String]
  (Player/deserialize (& bytes)))

;; Checked load (rejects malformed input)
(defn safe-load-player [(bytes : [Vec u8])] -> [Result Player DeserError]
  (Player/try-deserialize (& bytes)))
```

### Persistence to a File

```lisp
#[external]
(defn save-to-file [(state : Player)] -> [Result () String]
  (let bytes (serialize (& state)))
  (fs/write-bytes "save.dat" bytes))

#[external]
(defn load-from-file [] -> [Result Player String]
  (let bytes (? (fs/read-bytes "save.dat")))
  (Player/deserialize (& bytes)))
```

### Migration Example (Filling a Target Field)

```lisp
;; Source schema (stored data)
;; (defstruct PlayerState (hp : i32) (mp : i32))

;; Target schema
(defstruct PlayerState
  (hp  : i32)
  (mp  : i32)
  (exp : i64))   ;; added field

;; Read bytes from the source schema: exp is filled with 0
(defn load-state [(bytes : [Vec u8])] -> [Result PlayerState String]
  (PlayerState/deserialize (& bytes)))
```

### Consecutive Raw-BBin Writes and Reads

```lisp
(defstruct Header (kind : u8) (length : u32))
(defstruct Payload (x : f32) (y : f32))

(let mut bytes (Vec/new))
(Header/bbin-write (& header) (& mut bytes))
(Payload/bbin-write (& payload) (& mut bytes))

(let mut pos 0)
(match (Header/bbin-read (& bytes) (& mut pos))
  [(Ok decoded-header)
    (match (Payload/bbin-read (& bytes) (& mut pos))
      [(Ok decoded-payload) (consume decoded-header decoded-payload)]
      [(Err error) (handle-invalid-packet error)])]
  [(Err error) (handle-invalid-packet error)])
```

Before sending or receiving this format, verify that each type's `schema-id` matches the peer's.

---

## Performance Notes

- `T/serialize` returns `[Vec u8]`; reallocation may occur as it grows.
- `header_size` is a compile-time constant. Data is written directly to a single output `Vec`, without creating an intermediate `Vec` for the body.
- The fast path performs neither descriptor parsing nor string matching.
- The migration path is slower than the fast path because it matches type descriptors and converts the body.
- Raw BBin neither generates nor transfers a header or descriptor, but the caller must establish schema agreement.
- Bulk `memcpy` optimization for scalar `[Vec T]` values such as `[Vec i32]` is currently unimplemented; an element loop is used.

---

## Relationship to `persist`, `thaw`, and `freeze`

`persist`, `thaw`, and `freeze` are type-conversion operations for transferring and converting data structures between regions; they are independent of BBin serialization. BBin handles serialization to byte sequences for file storage, network transfer, and similar purposes, while persist/thaw handles conversion of arena-resident object graphs.
