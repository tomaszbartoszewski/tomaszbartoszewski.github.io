---
layout: post
title:  "Avro Binary encoding based on messages in Kafka"
date:   2020-07-12 10:00:00 +0000
categories: [avro, binary, encoding]
tags: [avro, binary, encoding, kafka]
---

In this post I will explore Avro binary encoding, we will go from simple one value schemas all the way to complex records. If you want to understand how values are converted into binary, or you want to manually read data from binary, this is a place for you.

## What I will cover here

In this post I will show you how data is encoded based on Avro schema. You will see how you can read data from binary using the schema to understand it. I'm using Kafka for examples but if you are only interested in Avro, that is the main content of this article.

## Useful resources

To learn about Avro binary encoding and how to read messages from Kafka in binary format I used few resources. Special thanks go to my co-worker Baptiste for hints where to look and what tools are useful.

[Kafka wire format](https://docs.confluent.io/current/schema-registry/serdes-develop/index.html#wire-format)

[Avro encoding](https://avro.apache.org/docs/1.7.7/spec.html#binary_encoding)

[Kafka REST](https://docs.confluent.io/current/kafka-rest/quickstart.html)

[Schema registry API](https://docs.confluent.io/current/schema-registry/develop/api.html)

My inspiration to understand binary encoding came from a book: Designing Data-Intensive Applications, Chapter 4

## Requirements

I used tools available on Ubuntu and MacOS, you may have to install them if you want to try it yourself, some may be already on your operating system - curl, base64, od, xxd.

* Docker
* Docker Compose
* [Code from GitHub](https://github.com/tomaszbartoszewski/avro-kafka-binary-encoding)
* curl
* base64
* od
* xxd
* [jq](https://stedolan.github.io/jq/)

## Kafka wire format

You can find this information [here](https://docs.confluent.io/current/schema-registry/serdes-develop/index.html#wire-format).
For convenience I copied it, we will ignore byte 0, then we can use following 4 to get schema id. We know what schema we used for messages in examples here, so we could ignore it, but I will show you how to use it. Bytes from 5 onwards is what we are interested in, it's the data.

Kafka wire format contains:

|Bytes|Area|Description|
|-|-|-|
|0|Magic Byte|Confluent serialization format version number; currently always 0.|
|1-4|Schema ID|4-byte schema ID as returned by Schema Registry.|
|5-…|Data|Serialized data for the specified schema format (for example, binary encoding for Avro or Protocol Buffers). The only exception is raw bytes, which will be written directly without any special encoding.|

## Avro schema

There are two formats, Avro IDL and JSON representation.

Avro IDL:
```
record User {
    string name;
    union { null, int } yearOfBirth = null;
    array<string> roles;
}
```

Avro JSON:
```
{
    "type": "record",
    "name": "User",
    "fields": [
        {
            "name": "name",
            "type": "string"
        },
        {
            "name": "yearOfBirth",
            "type": [
                "null",
                "int"
            ],
            "default": null
        },
        {
            "name": "roles",
            "type": {
                "type": "array",
                "items": "string"
            }
        }
    ]
}
```

Kafka REST uses JSON format, and it's what I will use in this post.

## Avro binary encoding

### Primitive Types

#### Null

Zero bytes, it may look a bit suspicious, but at the point of reading you know what type to expect, you will see it later when we discuss unions.

#### Boolean

Single byte, 1 represents true, 0 represents false.

#### Int and Long

Values are written with variable-length zig-zag coding. This is interesting.
Zig-zag means that we go back and forth between negative and positive values. You can notice that last bit of first byte indicate the sign, 1 means minus. Variable-length allow us to keep numbers with small absolute value short, first bit indicates if there is something after.

|Value|Hex|Binary|
|-|-|-|
|0|00|`00000000`|
|-1|01|`00000001`|
|1|02|`00000010`|
|-2|03|`00000011`|
|2|04|`00000100`|
|-3|05|`00000101`|
|-64|7f|`01111111`|
|64|80 01|`10000000 00000001`|
|1337|f2 14|`11110010 00010100`|

Let's look at few numbers.

1 is represented as `00000010`

|Has next byte?|Value from first byte|Sign|
|`0`|`000001`|`0`|

It's a single byte, no minus, we convert `000001` to decimal and we get 1.

Sign is only a part of the first byte, all following bytes will use 7 bits for encoding value.

64 is represented as `10000000 00000001`

|Has next byte?|Value from first byte|Sign|Has next byte?|Value from second byte|
|`1`|`000000`|`0`|`0`|`0000001`|

We now concatenate it from the end, value from second byte goes first followed by value from first byte, so `0000001` with `000000`, that gives us `0000001000000` which is 64.

1337 is represented as `11110010 00010100`

|Has next byte?|Value from first byte|Sign|Has next byte?|Value from second byte|
|`1`|`111001`|`0`|`0`|`0010100`|

We do same concatenation like for 64. `0010100111001` is 1337.

#### Float

I will skip floats in this demo, as they are more complicated than int and long in terms of encoding.
Just for completeness I copied description from documentation.

A float is written as 4 bytes. The float is converted into a 32-bit integer using a method equivalent to [Java's floatToIntBits](https://docs.oracle.com/javase/6/docs/api/java/lang/Float.html#floatToIntBits%28float%29) and then encoded in little-endian format.

#### Double

Like a Float, I will skip Double in examples, and only include description from documentation.

A double is written as 8 bytes. The double is converted into a 64-bit integer using a method equivalent to [Java's doubleToLongBits](https://docs.oracle.com/javase/6/docs/api/java/lang/Double.html#doubleToLongBits%28double%29) and then encoded in little-endian format.

#### Bytes

Bytes are encoded by long representing length, followed by that many bytes of data.

#### String

Long followed by that many bytes of UTF-8 encoded character data.

"foo" -> `06 66 6f 6f`

`06` is Avro binary encoded long 3 - described [here](#int-and-long).

`66` is ASCII for f.

`6f` is ASCII for o.

`6f` is ASCII for o.

### Complex Types

#### Records

There is no extra information telling that something is a record. The encoding data is a concatenation of the encodings of its fields, in order they are declared.

For a schema.
```
{
    "type": "record", 
    "name": "test",
    "fields" : [
        {"name": "a", "type": "long"},
        {"name": "b", "type": "string"}
    ]
}
```

And a value.
```
{
    "a": 27, 
    "b": "foo"
}
```

Encoding will have value `36 06 66 6f 6f`.

Field a:

`36` is Avro binary encoded long 27.

Field b:

`06` is Avro binary encoded long 3.

`66` is ASCII for f.

`6f` is ASCII for o.

`6f` is ASCII for o.

Encoding of long is described [here](#int-and-long) and string [here](#string)

#### Enums

An int, representing the zero-based position of the symbol in the schema.
```
{
    "type": "enum",
    "name": "Foo",
    "symbols": ["A", "B", "C", "D"]
}
```

|Enum symbol|Int value|Binary encoded int|
|A|0|`00000000`|
|B|1|`00000010`|
|C|2|`00000100`|
|D|3|`00000110`|

#### Arrays

A series of blocks. Each block consists of a count value (long), followed by that many array items.
A block with count zero indicates the end of the array.
Each item is encoded per the array's item schema.
I will focus on simple encoding, but there is an option for fast skipping through data. If count is negative, it's absolute value is used, but following value is a size of the block, which allow you to jump to next block without extra decoding of values in the block, to know where it ends.

For a schema.
```
{
    "type": "array",
    "items": "long"
}
```

And an array.
```
[
    3,
    27
]
```

Encoding will be `04 06 36 00`.

`04` is Avro binary encoded long 2 - number of items in the block.

`06` is Avro binary encoded long 3.

`36` is Avro binary encoded long 27.

`00` end of array - block with count 0.

#### Maps

Similar to array, we get blocks where first value is count of items in the block. Each item is encoded as string key and value encoding defined in the schema. A block with count zero indicates the end of the map, same as array. Same as arrays, when you see negative block's count, real count is an absolute value, but following value is the size of the block to allow fast skipping through data.

For a schema.
```
{
    "type": "map",
    "values": "long"
}
```

And a map.
```
{
    "a": 1,
    "b": -1
}
```

Encoding will be `04  02  61  02  02  62  01  00`.

`04` is Avro binary encoded long 2 - number of items in the block.

First item in the block, string key followed by long value.

`02` is Avro binary encoded long 1, this is the length of the string.

`61` is ASCII for a.

`02` is Avro binary encoded long 1.

Second item in the block, string key followed by long value.

`02` is Avro binary encoded long 1, this is length of the string.

`62` is ASCII for b.

`01` is Avro binary encoded long -1.

That's the end of first block, start of next block is `00` which means the end of the map.

#### Unions

Long value indicating the zero-based position within the union of the schema of its value.
The value is then encoded per the indicated schema within the union.

For a schema, we have a record which has one field called valueA of a type union between null, int and string
```
{
    "type":"record",
    "name":"UnionExample",
    "fields": [
        {
            "name":"valueA",
            "type": [
                "null",
                "int",
                "string"
            ],
            "default": null
        }
    ]
}
```

Let's have a look at 3 different examples, union is inside a record, but as you saw, record doesn't add extra bytes.

First example, union field has value null.

```
{
    "valueA": null
}
```

Will be encoded as `00`.

`00` is Avro binary encoded long 0, this is position inside union. It is type null, so no byte is needed after.

Second example, union field has value 4, we have to encapsulate it inside {"int": 4}, to give a union branch information.

```
{
    "valueA": {"int": 4}
}
```

Will be encoded as `02  08`.

`02` is Avro binary encoded long 1, this is position inside union. It is type int, so we read int next.

`08` is Avro binary encoded int 4.

Third example, union field has value "C", similar to int, we have to encapsulate it inside {"string": "C"}.

```
{
    "valueA": {"string": "C"}
}
```

Will be encoded as `04  02  43`.

`04` is Avro binary encoded long 2, this is position inside union. It is type string, so we read string next.

`02` is Avro binary encoded long 1, that's length of the string

`43` is ASCII for C.

#### Fixed

Encoded with number of bytes declared in the schema.

### Decode these secret messages

If you want to check your understanding of Avro binary encoding, try to solve some of these exercises.

For schema.
```
{
    "type":"record",
    "name":"NullableInt",
    "fields": [
        {
            "name":"favoriteNumber",
            "type": ["null", "long"],
            "default": null
        }
    ]
}
```

Encoded value is `02  04`. What does it represent?

<details>
    <summary>Answer</summary>
    { "favoriteNumber": 2 }
</details>
<br/>

For schema above, how will this value be encoded?
```
{
    "favoriteNumber": {"long": 64}
}
```

<details>
    <summary>Answer</summary>
    02  80  01
</details>
<br/>

For schema.
```
{
    "type":"record",
    "name":"Person",
    "fields": [
        {
            "name":"name",
            "type": "string"
        },
        {
            "name":"age",
            "type": "int"
        },
        {
            "name": "eyesColour",
            "type": {
                "name": "enumEyesColour",
                "symbols": [
                    "amber",
                    "blue",
                    "brown",
                    "gray",
                    "green",
                    "hazel",
                    "red"
                ],
                "type": "enum"
            }
        }
    ]
}
```

What will be the encoding for provided record?
```
{
    "name": "Debra",
    "age": 56,
    "eyesColour": "blue"
}
```

<details>
    <summary>Answer</summary>
    0a  44  65  62  72  61  70  02
</details>
<br/>

For above schema can you decode `08  4a  6f  68  6e  86  01  0a`?

<details>
    <summary>Answer</summary>
    { "name": "John", "age": 67, "eyesColour": "hazel" }
</details>
<br/>

## Running examples locally

To test how it works we are going to run Kafka with Schema Registry and Kafka REST.

### Start Kafka

For all examples clone [Code from GitHub](https://github.com/tomaszbartoszewski/avro-kafka-binary-encoding).

First step is to start Kafka with Docker Compose.

```
docker-compose up
```

It takes a while to start all services.

### Kafka REST

All necessary REST queries are inside bash scripts, but if you want to run everything manually I will describe them here.