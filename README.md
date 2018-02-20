# Wire encoding for Golang

This software implements Go bindings for the Wire encoding protocol.  The goal
of the Wire encoding protocol is to be a simple language-agnostic encoding
protocol for rapid prototyping of blockchain applications.

This package also includes a compatible (and slower) JSON codec.

## Interfaces and concrete types

Wire is an encoding library that can handle interfaces (like protobuf "oneof")
well.  This is achieved by prefixing bytes before each "concrete type".

A concrete type is some non-interface value (generally a struct) which
implements the interface to be (de)serialized. Not all structures need to be
registered as concrete types -- only when they will be stored in interface type
fields (or interface type slices) do they need to be registered.

### Registering types

All interfaces and the concrete types that implement them must be registered.

```golang
wire.RegisterInterface((*MyInterface1)(nil), nil)
wire.RegisterInterface((*MyInterface2)(nil), nil)
wire.RegisterConcrete(MyStruct1{}, "com.tendermint/MyStruct1", nil)
wire.RegisterConcrete(MyStruct2{}, "com.tendermint/MyStruct2", nil)
wire.RegisterConcrete(&MyStruct3{}, "anythingcangoinhereifitsunique", nil)
```

Notice that an interface is represented by a nil pointer.

Structures that must be deserialized as pointer values must be registered with
a pointer value as well.  It's OK to (de)serialize such structures in
non-pointer (value) form, but when deserializing such structures into an
interface field, they will always be deserialized as pointers.

### How it works

All registered concrete types are encoded with leading 4 bytes (called "prefix
bytes"), even when it's not held in an interface field/element.  In this way,
Wire ensures that concrete types (almost) always have the same canonical
representation.  The first byte of the prefix bytes must not be a zero byte, so
there are 2^(8x4)-2^(8x3) possible values.

When there are 4096 types registered at once, the probability of there being a
conflict is ~ 0.2%. See https://instacalc.com/51189 for estimation.  This is
assuming that all registered concrete types have unique natural names (e.g.
prefixed by a unique entity name such as "com.tendermint/", and not
"mined/grinded" to produce a particular sequence of "prefix bytes").

TODO Update instacalc.com link with 255/256 since 0x00 is an escape.

Do not mine/grind to produce a particular sequence of prefix bytes, and avoid
using dependencies that do so.

Since 4 bytes are not sufficient to ensure no conflicts, sometimes it is
necessary to prepend more than the 4 prefix bytes for disambiguation.  Like the
prefix bytes, the disambiguation bytes are also computed from the registered
name of the concrete type.  There are 3 disambiguation bytes, and in binary
form they always precede the prefix bytes.  The first byte of the
disambiguation bytes must not be a zero byte, so there are 2^(8x3)-2^(8x2)
possible values.

```
// Sample Wire encoded binary bytes with 4 prefix bytes.
> [0xBB 0x9C 0x83 0xDD] [...]

// Sample Wire encoded binary bytes with 3 disambiguation bytes and 4
// prefix bytes.
> 0x00 <0xA8 0xFC 0x54> [0xBB 0x9C 0x83 0xDD] [...]
```

The prefix bytes never start with a zero byte, so the disambiguation bytes are
escaped with 0x00.

Notice that the 4 prefix bytes always immediately precede the binary encoding
of the concrete type.

### Computing prefix bytes

To compute the disambiguation bytes, we take `hash := sha256(concreteTypeName)`,
and drop the leading 0x00 bytes.

```
> hash := sha256("com.tendermint.consensus/MyConcreteName")
> hex.EncodeBytes(hash) // 0x{00 00 A8 FC 54 00 00 00 BB 9C 83 DD ...} (example)
```

In the example above, hash has two leading 0x00 bytes, so we drop them.

```
> rest = dropLeadingZeroBytes(hash) // 0x{A8 FC 54 00 00 BB 9C 83 DD ...}
> disamb = rest[0:3]
> rest = dropLeadingZeroBytes(rest[3:])
> prefix = rest[0:4]
```

The first 3 bytes are called the "disambiguation bytes" (in angle brackets).
The next 4 bytes are called the "prefix bytes" (in square brackets).

```
> <0xA8 0xFC 0x54> [0xBB 0x9C 9x83 9xDD]
```

### Supported types

**Primary types**: `uvarint`, `varint`, `byte`, `uint[8,16,32,64]`, `int[8,16,32,64]`, `string`, and `time` types are supported

**Arrays**: Arrays can hold items of any arbitrary type.  For example, byte-arrays and byte-array-arrays are supported.

**Structs**: Struct fields are encoded by value (without the key name) in the order that they are declared in the struct.  In this way it is similar to Apache Avro.

**Interfaces**: Interfaces are like union types where the value can be any non-interface type. The actual value is preceded by a single "type byte" that shows which concrete is encoded.

**Pointers**: Pointers are like optional fields.  The first byte is 0x00 to denote a null pointer (e.g. no value), otherwise it is 0x01.

### Unsupported types

**Maps**: Maps are not supported because for most languages, key orders are nondeterministic.
If you need to encode/decode maps of arbitrary key-value pairs, encode an array of {key,value} structs instead.

**Floating points**: Floating point number types are discouraged because [of reasons](http://gafferongames.com/networking-for-game-programmers/floating-point-determinism/).  If you need to use them, use the field tag `wire:"unsafe"`.

**Enums**: Enum types are not supported in all languages, and they're simple enough to model as integers anyways.

## Forward and Backward compatibility

TODO

## Wire vs JSON

TODO

## Wire vs Protobuf

From the [Protocol Buffers encoding guide](https://developers.google.com/protocol-buffers/docs/encoding):

> As you know, a protocol buffer message is a series of key-value pairs. The
> binary version of a message just uses the field's number as the key – the
> name and declared type for each field can only be determined on the decoding
> end by referencing the message type's definition (i.e. the .proto file).
>
> When a message is encoded, the keys and values are concatenated into a byte
> stream. When the message is being decoded, the parser needs to be able to
> skip fields that it doesn't recognize. This way, new fields can be added to a
> message without breaking old programs that do not know about them. To this
> end, the "key" for each pair in a wire-format message is actually two values
> – the field number from your .proto file, plus a wire type that provides just
> enough information to find the length of the following value.
>
> The available wire types are as follows:
> 
> Type | Meaning | Used For
> ---- | ------- | --------
> 0    | Varint  | int32, int64, uint32, uint64, sint32, sint64, bool, enum
> 1    | 64-bit  | fixed64, sfixed64, double
> 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields
> 3    | Start group | groups (deprecated)
> 4    | End group | groups (deprecated)
> 5    | 32-bit  | fixed32, sfixed32, float
>
> Each key in the streamed message is a varint with the value (field_number <<
> 3) | wire_type – in other words, the last three bits of the number store the
> wire type.

In Wire, 

Type | Meaning | Used For
---- | ------- | --------
0    | Varint  | bool, byte, [u]int16, and varint-[u]int[64/32]
1    | 64-bit  | int64, uint64, float64(unsafe)
2    | Length-delimited | string, bytes, raw?
3    | Start struct | conceptually, '{'
4    | End group | conceptually, '}'
5    | 32-bit  | int32, uint32, float32
6    | List    | array, slice; followed by `<type3><varint-length-bytes>`
7    | interface | followed by `<prefix>` or `<disfix>`

Struct fields are encoded in order, and a null field is represented by the
absence of a field in the encoding, similar to protobuf. In general, the total
byte-size of a Wire:binary encoded struct cannot be determined until each
field's size has determined recursively.

As in protobuf, each struct field is keyed by a varint with the value
`(field_number << 3) | type`, where `type` is 3 bits long. 

A List is encoded by first writing 1 byte to represent the element type, which
is one of the above, from 0 ~ 7.  This "element type byte" is followed by a
varint representation of the number of elements in the parent object.

An interface value is typically a struct, but it doesn't need to be.  If it
isn't, then it is followed by the byte `0000 1XXX` where `XXX` denote the type
3-bit sequence.  This encodes non-struct interface values as if they were the
only field of a struct.  This extra byte for non-structs is necessary to be
able to scan and parse the structure of a Wire:binary blob.


## Wire in other langauges

Coming soon...
