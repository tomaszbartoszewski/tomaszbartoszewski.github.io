---
layout: post
title:  "Avro Binary encoding based on messages in Kafka"
date:   2020-07-12 10:00:00 +0000
categories: [avro, binary, encoding]
tags: [avro, binary, encoding, kafka]
---

In this post we will explore Avro binary encoding. We will go from simple one value schemas, all the way to complex records. If you want to understand how values are encoded, or you want to manually read data from binary, this is a place for you.

## What I will cover here

In this post I will show you how encoding works for different types. We will go first through string, int and other primitive types. Next we will go through encoding of complex types such as records and unions. You will see how to read data from binary using the schema for decoding. I'm using Kafka for examples, but if you are only interested in Avro, that is the main content of this article.

## Useful resources

To learn about Avro binary encoding and how to read messages from Kafka in binary format, I used few resources. Special thanks go to my co-worker Baptiste for hints where to look and what tools are useful.

[Kafka wire format](https://docs.confluent.io/current/schema-registry/serdes-develop/index.html#wire-format)

[Avro encoding](https://avro.apache.org/docs/1.7.7/spec.html#binary_encoding)

[Kafka REST](https://docs.confluent.io/current/kafka-rest/quickstart.html)

[Schema registry API](https://docs.confluent.io/current/schema-registry/develop/api.html)

My inspiration to understand binary encoding came from a book: Designing Data-Intensive Applications, Chapter 4

## Requirements

I used tools available on Ubuntu and MacOS, you may have to install them if you want to try it yourself. You may have some available on your operating system - curl, base64, od, xxd.

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

Zero bytes, it may look a bit suspicious, but at the point of reading you know what type to expect. It will make more sense when we discuss unions.

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

We concatenate it from the end, value from second byte goes first followed by value from first byte. `0000001` with `000000`, that gives us `0000001000000` which is 64.

1337 is represented as `11110010 00010100`

|Has next byte?|Value from first byte|Sign|Has next byte?|Value from second byte|
|`1`|`111001`|`0`|`0`|`0010100`|

We do same concatenation like for 64. `0010100111001` is 1337.

#### Float

I will skip floats in this demo, as they are more complicated than int and long in terms of encoding.
Just for completeness I copied the description from the documentation.

A float is written as 4 bytes. The float is converted into a 32-bit integer using a method equivalent to [Java's floatToIntBits](https://docs.oracle.com/javase/6/docs/api/java/lang/Float.html#floatToIntBits%28float%29) and then encoded in little-endian format.

#### Double

Like a Float, I will skip Double in examples, and only include the description from the documentation.

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

Long value indicating the zero-based position within the union of the schema of its values.
The value is then encoded per the indicated schema within the union.

For a schema, we have a record which has one field called valueA of a type union between null, int and string.
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

`02` is Avro binary encoded long 1, that's the length of the string

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

All necessary REST queries are inside bash scripts, but if you want to run everything manually, I will describe them here.

To publish messages to Kafka topic `user_data` with avro schema you have to pass two headers `Content-Type: application/vnd.kafka.avro.v2+json` and `Accept: application/vnd.kafka.v2+json`. Data is a JSON object which has `key_schema`, `value_schema` and `records`. Key schema and value schema are strings representing avro schemas in JSON format. Records is an array of JSON objects containing `key` and `value`. It's recommended to specify a key when publishing to Kafka, to make sure messages of same key are published to the same partition. I will skip key for simplicity, this way you don't have to give `key_schema` and `key`.

```
curl -X POST -H "Content-Type: application/vnd.kafka.avro.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '
      {
          "value_schema": "{\"type\": \"record\", \"name\": \"User\", \"fields\": [{\"name\": \"name\", \"type\": \"string\"}]}",
          "records": [
              {
                  "value": {"name": "Aragorn"}
              },
              {
                  "value": {"name": "Galadriel"}
              },
              {
                  "value": {"name": "Gimli"}
              },
              {
                  "value": {"name": "Arwen"}
              }
          ]
      }' \
      "http://localhost:8082/topics/user_data"

```

To read the messages we have to create a consumer. If you specify format as `avro` you will see messages in human readable format. To see binary data set it to `binary`.

```
curl -s -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '
      {
          "name": "user_data_consumer_instance",
          "format": "binary",
          "auto.offset.reset": "earliest"
      }' \
      http://localhost:8082/consumers/user_data_binary_consumer
```

Set from which topics consumer has to read.

```
curl -s -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '{"topics":["user_data"]}' \
      http://localhost:8082/consumers/user_data_binary_consumer/instances/user_data_consumer_instance/subscription
```

Read messages.

```
curl -s -X GET -H "Accept: application/vnd.kafka.binary.v2+json" \
      http://localhost:8082/consumers/user_data_binary_consumer/instances/user_data_consumer_instance/records
```

If you run this commands locally you should see messages. If it's the first time since Kafka started, you may get empty array. Send last request again to consume records.

The output should look like this.
```
[
    {
        "topic":"user_data",
        "key":null,
        "value":"AAAAAAEOQXJhZ29ybg==",
        "partition":0,
        "offset":0
    },
    {
        "topic":"user_data",
        "key":null,
        "value":"AAAAAAESR2FsYWRyaWVs",
        "partition":0,
        "offset":1
    },
    {
        "topic":"user_data",
        "key":null,
        "value":"AAAAAAEKR2ltbGk=",
        "partition":0,
        "offset":2
    },
    {
        "topic":"user_data",
        "key":null,
        "value":"AAAAAAEKQXJ3ZW4=",
        "partition":0,
        "offset":3
    }
]
```

Let's look at first value, it may be different for you, if the schema has different id.

```
echo "AAAAAAEOQXJhZ29ybg==" | base64 --decode | od -t x1 -Ad
```

That will give you this output.

```
0000000    00  00  00  00  01  0e  41  72  61  67  6f  72  6e
0000013
```

First byte is a magic byte, next 4 bytes represent the schema id. `00  00  00  01` means 1, if you query Schema Registry, you should see schema used for this message.

```
curl -s -X GET http://0.0.0.0:8081/schemas/ids/1
```

You can pipe it to jq, to see schema formatted.

```
curl -s -X GET http://0.0.0.0:8081/schemas/ids/1 | jq -r '.schema' | jq .
```

It will return a schema.

```
{
  "type": "record",
  "name": "User",
  "fields": [
    {
      "name": "name",
      "type": "string"
    }
  ]
}
```

We know that the schema is a record containing one string, that means that we expect encoding for a string. `0e` is a long encoding for 7, length of the string. Text is encoded as `41  72  61  67  6f  72  6e`, which is hex ASCII representation of `Aragorn`.

Last step is to delete the consumer.

```
curl -X DELETE -H "Content-Type: application/vnd.kafka.v2+json" \
      http://localhost:8082/consumers/user_data_binary_consumer/instances/user_data_consumer_instance
```

You can look at [Code from GitHub](https://github.com/tomaszbartoszewski/avro-kafka-binary-encoding) to see more scripts producing and consuming avro messages with different schemas. Look at messages and a schema, try to decode them yourself.

### Complex object

This is an example from `complex_object.sh`. It produces a message with a schema.

```
{
    "type": "record",
    "name": "FactoryWorker",
    "fields": [
        {
            "name": "employeeId",
            "type": "string"
        },
        {
            "name": "employeeRole",
            "type": {
                "name": "enumType",
                "symbols": [
                    "Manager",
                    "ProductionWorker"
                ],
                "type": "enum"
            }
        },
        {
            "name": "team",
            "type": [
                "null",
                "string"
            ],
            "default": null
        },
        {
            "name": "weeklyProduction",
            "type": {
                "items": {
                    "fields": [
                        {
                            "name": "timeStart",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "timeEnd",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "manufactured",
                            "type": "int"
                        }
                    ],
                    "name": "DailyProduction",
                    "type": "record"
                },
                "type": "array"
            }
        },
        {
            "name": "workSchedule",
            "type": [
                "null",
                {
                    "name": "Schedule",
                    "type": "record",
                    "fields": [
                        {
                            "name": "monday",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "tuesday",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "wednesday",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "thursday",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "friday",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "saturday",
                            "type": [
                                "null",
                                "string"
                            ]
                        },
                        {
                            "name": "sunday",
                            "type": [
                                "null",
                                "string"
                            ]
                        }
                    ]
                }
            ]
        }
    ]
}

```

You can see a message in base64. You may have different value, if your schema id is different.

```
"AAAAAANIYWEzYWZkNDQtODExOS00NTY1LTgwOWItOWRmNzVkNmRkYWIwAgICQgQCCDA3MDACCDE1MDAYAggxMjAwAggyMDAwEAACAggwMTAwAggwMjAwAggwMzAwAggwNDAwAggwNTAwAggwNjAwAggwNzAw"
```

To get hex value run.

```
echo "AAAAAANIYWEzYWZkNDQtODExOS00NTY1LTgwOWItOWRmNzVkNmRkYWIwAgICQgQCCDA3MDACCDE1MDAYAggxMjAwAggyMDAwEAACAggwMTAwAggwMjAwAggwMzAwAggwNDAwAggwNTAwAggwNjAwAggwNzAw"\
| base64 --decode | od -t x1 -Ad
```

The message published has this hex value.

```
00  00  00  00  03  48  61  61  33  61  66  64  34  34  2d  38
31  31  39  2d  34  35  36  35  2d  38  30  39  62  2d  39  64
66  37  35  64  36  64  64  61  62  30  02  02  02  42  04  02
08  30  37  30  30  02  08  31  35  30  30  18  02  08  31  32
30  30  02  08  32  30  30  30  10  00  02  02  08  30  31  30
30  02  08  30  32  30  30  02  08  30  33  30  30  02  08  30
34  30  30  02  08  30  35  30  30  02  08  30  36  30  30  02
08  30  37  30  30
```

We can skip first 5 bytes, as we know the schema.
First field is a string, `48` is a long encoding for 36, that's a length of the string, `61  61  33  61  66  64  34  34  2d  38 31  31  39  2d  34  35  36  35  2d  38  30  39  62  2d  39  64 66  37  35  64  36  64  64  61  62  30`
represents `aa3afd44-8119-4565-809b-9df75d6ddab0`.

Next field is an enum, we have `02` that's encoded int 1, so second value of an enum, which is `ProductionWorker`.

Next field is a union of a null and a string. `02` is encoded long 1, which means it's second type from a union, a string in this case. `02` represents a length 1, which gives `42` as a string, which is `B`.

Next we have an array, first block has length `04` which is encoded 2. It is a record of 2 unions and an int. `02` means 1, second type of union that's string. `08` length 4, `30  37  30  30` represents `0700`. Next union `02` means 1, string again. Length `08` which is 4, `31  35  30  30` represents `1500`. Then `18` means int 12. First record done. I will not describe the process, as it's the same. It covers bytes `02  08  31  32  30  30  02  08  32  30  30  30  10`, that's `1200`, `2000` and 8. Next block has length `00` which means the end of an array.

Last field is a union of null and a record of 7 fields which are unions of a null and a string. `02` means we have a record. You already know how to decode unions of nulls with strings, so I will just write them down in a list, `02` stands for 1, second union type which is string, `08` stands for 4, length of the string. All fields are strings of 4 characters, hence all start with `02  08`.

`02  08  30  31  30  30` - `0100`

`02  08  30  32  30  30` - `0200`

`02  08  30  33  30  30` - `0300`

`02  08  30  34  30  30` - `0400`

`02  08  30  35  30  30` - `0500`

`02  08  30  36  30  30` - `0600`

`02  08  30  37  30  30` - `0700`

That's the end of the message, you can compare it with avro consumer to check that it's matching.

## Summary

Hopefully now you will find binary encoding a bit less intimidating. Don't worry if something is still not clear, try to modify schema and messages to see how binary representation is changing. Try with small schemas first, as once you understand how they work, it will be easier to read more complex examples. If you are still interested in this topic, you can try to explore how Float and Double are represented. Or even try to implement your own encoder or decoder.