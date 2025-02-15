---
uid: sdsWritingData
---

# Write data

The SDS REST APIs provide programmatic access to write data to SDS.

All writes rely on a stream’s key or primary index. The primary index determines the order of events in the stream. Secondary indexes are updated, but they do not contribute to the request. All references to indexes are to the primary index.

## Single stream writes

The following methods support writing a single or multiple values:

* [Insert Values](xref:sdsWritingDataApi#insert-values) inserts a collection of events.
* [Patch Values](xref:sdsWritingDataApi#patch-values) updates specific fields for a collection of events.
* [Replace Values](xref:sdsWritingDataApi#replace-values) replaces a collection of events.
* [Remove Values](xref:sdsWritingDataApi#remove-values) deletes the events based on the request parameters.
* [Update Values](xref:sdsWritingDataApi#update-values) add or replaces a collection of events.

The base URI for writing SDS data to a single stream is:

```text
api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data  
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

## Request body format

With the exception of ``Remove Values``, all single stream write calls require a request body containing the events to insert or modify.
The events must be formatted as a serialized JSON array of the stream's type. JSON arrays are comma-delimited lists of a type enclosed within square brackets. The following code shows a list of three WaveData events that are properly formatted for insertion. For the complete example, see the [OCS-Samples](https://github.com/osisoft/OCS-Samples).

```json
[
    {
        "Order":2,
        "Tau":0.25722883666666846,
        "Radians":1.6162164471269089,
        "Sin":1.9979373673043652,
        "Cos":-0.090809010174665111,
        "Tan":-44.003064529862513,
        "Sinh":4.8353589272389,
        "Cosh":5.2326566823391856,
        "Tanh":1.8481468289554672
    },
    {
        "Order":4,
        "Tau":0.25724560000002383,
        "Radians":1.6163217742567466,
        "Sin":1.9979277915696148,
        "Cos":-0.091019446679060964,
        "Tan":-43.901119254534827,
        "Sinh":4.8359100947709592,
        "Cosh":5.233166005842703,
        "Tanh":1.8481776000882766
    },
    {
        "Order":6,
        "Tau":0.25724560000002383,
        "Radians":1.6163217742567466,
        "Sin":1.9979277915696148,
        "Cos":-0.091019446679060964,
        "Tan":-43.901119254534827,
        "Sinh":4.8359100947709592,
        "Cosh":5.233166005842703,
        "Tanh":1.8481776000882766
    }
]
```

You can serialize your data using one of many available JSON serializers available at [Introducing JSON](http://json.org/index.html).

## Indexes

SDS writes rely on the primary index for positioning within streams and locating existing events. Most writes use the index as specified by the value; however, for deletes, indexes are specified as strings in the URI. For more details about working with indexes, see [Indexes](xref:sdsIndexes).

To specify compound indexes in the URI, specify each field that composes the index, in the specified order, separated by the pipe character, ‘|’.
  