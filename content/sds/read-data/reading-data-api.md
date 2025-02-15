---
uid: sdsReadingDataApi
---

# Read data API

The following example API calls show different methods for reading data.

## Example type, stream, and data

Many of the API methods described below contain example requests and responses in JSON to highlight usage and specific behaviors. The following type, stream, and data are used in the examples:

**Example type**  
``SimpleType`` is an SdsType with a single index. This type is defined in Python and Javascript:

###### Python

```python
class State(Enum):
  Ok = 0
  Warning = 1
  Alarm = 2

class SimpleType(object):
  Time = property(getTime, setTime)
  def getTime(self):
    return self.__time
  def setTime(self, time):
    self.__time = time

  State = property(getState, setState)
  def getState(self):
    return self.__state
  def setState(self, state):
    self.__state = state

  Measurement = property(getValue, setValue)
  def getValue(self):
    return self.__measurement
  def setValue(self, measurement):
    self.__measurement = measurement
```

###### JavaScript

```javascript
var State =
{
  Ok: 0,
  Warning: 1,
  Alarm: 2,
}

var SimpleType = function () {
  this.Time = null;
  this.State = null;
  this.Value = null;
}
```

**Example stream**  
``Simple`` is an SdsStream of type ``SimpleType``.

**Example data**  
``Simple`` has stored values as follows:
    11/23/2017 12:00:00 PM: Ok  0
    11/23/2017  1:00:00 PM: Ok 10
    11/23/2017  2:00:00 PM: Ok 20
    11/23/2017  3:00:00 PM: Ok 30
    11/23/2017  4:00:00 PM: Ok 40

All times are represented at offset 0, GMT.
*****

## ``Get First Value``

Returns the first value in the stream. If no values exist in the stream, null is returned.

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data/First
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

**Response**  
The response includes a status code and a response body containing a serialized event.
*****

## ``Get Last Value``

Returns the last value in the stream. If no values exist in the stream, null is returned.

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data/Last
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

**Response**  
The response includes a status code and a response body containing a serialized event.
*****

## ``Find Distinct Value``

Returns a stored event based on the specified `index` and `searchMode`.

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data
    ?index={index}&searchMode={searchMode}
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string index``  
The index.

``string searchMode``  
The [SdsSearchMode](xref:Enums#sdssearchmode); the default is ``exact``.

**Response**  
The response includes a status code and a response body containing a serialized collection with one event. Depending on the request `index` and `searchMode`, it is possible to have an empty collection returned.

**Example**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?index=2017-11-23T13:00:00Z&searchMode=Next
```

The request has an index that matches the index of an existing event, but since a `SdsSearchMode` of ``next`` was specified, the response contains the next event in the stream after the specified index:

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    }
]
```

**Example**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?index=2017-11-23T13:30:00Z&searchMode=Next
```

The request specifies an index that does not match an index of an existing event. The next event in the stream is retrieved.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    }
]
```

*****

## ``Get Values``

Returns a collection of *stored* values at indexes based on request parameters.
SDS supports three ways of specifying which stored events to return:

* [Filtered](#getvaluesfiltered): A filtered request accepts a filter expression.
* [Range](#getvaluesrange): A range request accepts a start index and a count.
* [Window](#getvalueswindow): A window request accepts a start index and end index. This request has an optional continuation token for large collections of events.

<a name="getvaluesfiltered"></a>

### `Filtered`  

Returns a collection of stored values as determined by a `filter`. The `filter` limits results by applying an expression against event fields. For more information about filter expressions, see [Filter expressions](xref:sdsFilterExpressions).

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data
    ?filter={filter}
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string filter``  
The filter expression (see [Filter expressions](xref:sdsFilterExpressions)).

**Response**  
The response includes a status code and a response body containing a serialized collection of events.

**Example**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?filter=Measurement gt 10
```

The events in the stream with `Measurement` greater than 10 are returned.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T14:00:00Z",
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "Measurement": 30
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "Measurement": 40
    }
]
```

**Note:** `State` is not included in the JSON as its value is the default value.

<a name="getvaluesrange"></a>

### `Range`

Returns a collection of stored values as determined by a ``startIndex`` and ``count``. Additional optional parameters specify the direction of the range, how to handle events near or at the start index, whether to skip a certain number of events at the start of the range, and how to filter the data.

**Request**

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data
    ?startIndex={startIndex}&count={count}[&skip={skip}&reversed={reversed}
    &boundaryType={boundaryType}&filter={filter}]
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string startIndex``  
Index identifying the beginning of the series of events to return.

``int count``  
The number of events to return.

``int skip``  
Optional value specifying the number of events to skip at the beginning of the result.

``bool reversed``  
Optional specification of the direction of the request. By default, range requests move forward from startIndex, collecting events after startIndex from the stream. A reversed request will collect events before startIndex from the stream.

``SdsBoundaryType boundaryType``  
Optional SdsBoundaryType specifies the handling of events at or near startIndex.

``string filter``  
Optional filter expression.

**Response**  
The response includes a status code and a response body containing a serialized collection of events.

**Example**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T13:00:00Z&count=100
```

This request will return a response with up to 100 events starting at 13:00 and extending forward toward the end of the stream:

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "Measurement": 30
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "Measurement": 40
    }
]
```

**Note:** `State` is not included in the JSON as its value is the default value.

**Example**  
To reverse the direction of the request, set reversed to true. The following request will return up to 100 events starting at 13:00 and extending back toward the start of the stream:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T13:00:00Z&count=100&reversed=true
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T12:00:00Z"
    }
]
```

**Note:** `State` is not included in the JSON as its value is the default value.

Further, `Measurement` is not included in the second, 12:00:00, event as zero is the default value for numbers.

The following request specifies a boundary type of Outside for a reversed-direction range request. The response will contain up to 100 events. The boundary type Outside indicates that up to one event outside the boundary will be included in the response. For a reverse direction range request, this means one event forward of the specified start index. In a default direction range request, it would mean one event before the specified start index.

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T13:00:00Z&count=100&reversed=true&boundaryType=2
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T12:00:00Z",
        "State": 0,
        "Measurement": 0
    }
]
```

The event outside of the index is the next event or the event at 14:00 because the request operates in reverse.

Adding a filter to the request means only events that meet the filter criteria are returned:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T13:00:00Z&count=100&reversed=true&boundaryType=2&filter=Measurement gt 10
 ```

**Response body**
  
```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    }
]
```

<a name="getvalueswindow"></a>

### `Window`

Returns a collection of stored events based on the specified `startIndex` and `endIndex`.

For handling events at and near the boundaries of the window, a single SdsBoundaryType that applies to both the start and end indexes can be passed with the request, or separate boundary types may be passed for the start and end individually.

Paging is supported for window requests with a large number of events.

To retrieve the next page of values, include the `continuationToken` from the results of the previous request. For the first request, specify a null or empty string for the `continuationToken`.

**Requests**

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data?
    ?startIndex={startIndex}&endIndex={endIndex}

GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data?
    ?startIndex={startIndex}&endIndex={endIndex}&boundaryType={boundaryType}

GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data?
    ?startIndex={startIndex}&startBoundaryType={startBoundaryType}
    &endIndex={endIndex}&endBoundaryType={endBoundaryType}

GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data?
    ?startIndex={startIndex}&endIndex={endIndex}
    &count={count}&continuationToken={continuationToken}

GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data?
    ?startIndex={startIndex}&startBoundaryType={startBoundaryType}
    &endIndex={endIndex}&endBoundaryType={endBoundaryType}&filter={filter}&count={count}
    &continuationToken={continuationToken}
 ```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string startIndex``  
Index bounding the beginning of the series of events to return.

``string endIndex``  
Index bounding the end of the series of events to return.

``int count``  
Optional maximum number of events to return. If `count` is specified, a `continuationToken` must also be specified.

``SdsBoundaryType boundaryType``  
Optional [SdsBoundaryType](xref:Enums#sdsboundarytype) specifies handling of events at or near the start and end indexes.

``SdsBoundaryType startBoundaryType``  
Optional [SdsBoundaryType](xref:Enums#sdsboundarytype) specifies the first value in the result in relation to the start index. If `startBoundaryType` is specified, `endBoundaryType` must be specified.

``SdsBoundaryType endBoundaryType``  
Optional [SdsBoundaryType](xref:Enums#sdsboundarytype) specifies the last value in the result in relation to the end index. If `startBoundaryType` is specified, `endBoundaryType` must be specified.

``string filter``  
Optional filter expression (see [Filter expressions](xref:sdsFilterExpressions)).

``string continuationToken``  
Optional token used to retrieve the next page of data. If `count` is specified, a `continuationToken` must also be specified.

**Response**  
The response includes a status code and a response body containing a serialized collection of events. A continuation token can be returned if specified in the request.

**Example**  
The following requests all stored events between 12:30 and 15:30:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T12:30:00Z&endIndex=2017-11-23T15:30:00Z
```

The response will contain the event stored at the specified index:

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "Measurement": 30
    }
]
```

**Note:** `State` is not included in the JSON as its value is the default value.

**Example**  
When the request is modified to specify a boundary type of Outside, the value before 13:30 and the value after 15:30 are included:
  
```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T12:30:00Z&endIndex=2017-11-23T15:30:00Z
    &boundaryType=2
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T12:00:00Z"
    },
    {
        "Time": "2017-11-23T13:00:00Z",
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "Measurement": 30
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "Measurement": 40
    }
]
```

**Note:** `State` is not included in the JSON as its value is the default value. Further, `Measurement` is not included in the second, 12:00:00, event as zero is the default value for numbers.

If instead a start boundary of Inside, only values inside the start boundary (after 13:30) are included in the result. With an end boundary of Outside one value outside the end index (after 15:30) is included:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T12:30:00Z&&startBoundaryType=1
    &endIndex=2017-11-23T15:30:00Z&endBoundaryType=2
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "State": 0,
        "Measurement": 30
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "State": 0,
        "Measurement": 40
    }
]
```

To page the results of the request, a continuation token may be specified. This requests the first page of the first two stored events between start index and end index by indicating count is 2 and continuationToken is an empty string:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T12:30:00Z&endIndex=2017-11-23T15:30:00Z
    &count=2&continuationToken=
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

{
    "Results": [
        {
            "Time": "2017-11-23T13:00:00Z",
            "State": 0,
            "Measurement": 10
        },
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 20
        }
    ],
    "ContinuationToken": "2017-11-23T14:00:00.0000000Z"
}
```

This request uses the continuation token from the previous page to request the next page of stored events:

```text
GET api/v1/Tenants/default}/Namespaces/{namespaceId}/Streams/Simple/Data
    ?startIndex=2017-11-23T12:30:00Z&endIndex=2017-11-23T15:30:00Z
    &count=2&continuationToken=2017-11-23T14:00:00Z
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

{
    "Results": [
        {
            "Time": "2017-11-23T15:00:00Z",
            "State": 0,
            "Measurement": 30
        }
    ],
    "ContinuationToken": null
}
```

In this case, the results contain the final event. The returned continuation token is null.
*****

## `Get Interpolated Values`

Returns a collection of values based on request parameters. The stream's read characteristics determine how events are calculated for indexes at which no stored event exists. Interpolation is not supported for streams with compound indexes.
SDS supports two ways of specifying which interpolated events to return:  

* [Index Collection](#getvaluesindexcollection): One or more indexes can be passed to the request to retrieve events at specific indexes.
* [Interval](#getvaluesinterpolatedinterval): An interval can be specified with a start index, end index, and count. This will return the specified count of events evenly spaced from start index to end index.

<a name="getvaluesindexcollection"></a>

### `Index Collection`  

Returns events at the specified indexes. If no stored event exists at a specified index, the stream's read characteristics determine how the returned event is calculated.

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data/
    Interpolated?index={index}[&index={index}...]
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string index``  
One or more indexes.

**Response**  
The response includes a status code and a response body containing a serialized collection of events. Depending on the specified indexes and read characteristics of the stream, it is possible to have less events returned than specified indexes. An empty collection can also be returned.

**Example**  
Consider a stream of type ``Simple`` with the default ``InterpolationMode`` of ``Continuous`` and ``ExtrapolationMode`` of ``All``. In the following request, the specified index matches an existing stored event:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data/
    Interpolated?index=2017-11-23T13:00:00Z
```

The response will contain the event stored at the specified index.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 10
    }
]
```

The following request specifies an index for which no stored event exists:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data/
    Interpolated?index=2017-11-23T13:30:00Z
```

Because the index is a valid type for interpolation and the stream has an ``InterpolationMode`` of ``Continuous``, this request receives a response with an event interpolated at the specified index:

**Response body**
  
```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:30:00Z",
        "State": 0,
        "Measurement": 15
    }
]
```

Consider a stream of type ``Simple`` with an ``InterpolationMode`` of ``Discrete`` and ``ExtrapolationMode`` of ``All``. In the following request, the specified indexes only match two existing stored events:

```text
GET api/v1/Tenants/default}/Namespaces/{namespaceId}/Streams/Simple/Data
    Interpolated?index=2017-11-23T12:30:00Z&index=2017-11-23T13:00:00Z&index=2017-11-23T14:00:00Z
```

For this request, the response contains events for two of the three specified indexes.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    }
]
```

<a name="getvaluesinterpolatedinterval"></a>

### `Interval`

Returns events at evenly spaced intervals based on the specified start index, end index, and count. If no stored event exists at an index interval, the stream's read characteristics determine how the returned event is calculated.

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data/
    Interpolated?startIndex={startIndex}&endIndex={endIndex}&count={count}
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string startIndex``  
The index defining the beginning of the window.

``string endIndex``  
The index defining the end of the window.  

``int count``  
The number of events to return. Read characteristics of the stream determine how the events are constructed.

**Response**  
The response includes a status code and a response body containing a serialized collection of events. Depending on the read characteristics and input parameters, it is possible for a collection to be returned with less events than specified in the count.

For a stream, named Simple, of type ``Simple`` for the following request:

```text
GET api/v1/Tenants/default}/Namespaces/{namespaceId}/Streams/Simple/Data/
    Interpolated?startIndex=2017-11-23T13:00:00Z&endIndex=2017-11-23T15:00:00Z&count=3
```

The start and end fall exactly on event indexes, and the number of events from start to end match the count of three (3).

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "State": 0,
        "Measurement": 30
    }
]
```

*****

## ``Get Summaries``

Returns summary intervals between a specified start and end index.

Index types that cannot be interpolated do not support summary requests. Strings are an example of indexes that cannot be interpolated. Summaries are not supported for streams with compound indexes. Interpolating between two indexes that consist of multiple properties is not defined and results in non-determinant behavior.

Summary values supported by `SdsSummaryType` enum:

| Summary                            | Enumeration value |
| ---------------------------------- | ----------------- |
| Count                              | 1                 |
| Minimum                            | 2                 |
| Maximum                            | 4                 |
| Range                              | 8                 |
| Mean                               | 16                |
| StandardDeviation                  | 64                |
| Total                              | 128               |
| Skewness                           | 256               |
| Kurtosis                           | 512               |
| WeightedMean                       | 1024              |
| WeightedStandardDeviation          | 2048              |
| WeightedPopulationStandardDeviatio | 4096              |

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data/
    Summaries?startIndex={startIndex}&endIndex={endIndex}&count={count}&filter={filter}
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string startIndex``  
The start index for the intervals.

``string endIndex``  
The end index for the intervals.

``int count``  
The number of intervals requested.

``string filter``  
Optional filter expression (see [Filter expressions](xref:sdsFilterExpressions)).

``string streamViewId``  
Optional stream view identifier.

**Response**  
The response includes a status code and a response body containing a serialized collection of SdsIntervals.

Each SdsInterval has a start, end, and collection of summary values.

| Property  | Details                                           |
| --------- | ------------------------------------------------- |
| Start     | The start of the interval                         |
| End       | The end of the interval                           |
| Summaries | The summary values for the interval, keyed by summary type. The nested dictionary contains property name keys and summary calculation result values. |

**Example**  
The following request calculates two summary intervals between the `startIndex` and `endIndex`:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data/ 
    Summaries?startIndex=2017-11-23T12:00:00Z&endIndex=2017-11-23T16:00:00Z&count=2
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Start": {
            "Time": "2017-11-23T12:00:00Z",
            "State": 0,
            "Measurement": 0
        },
        "End": {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 20
        },
        "Summaries": {
            "Count": {
                "Measurement": 2
            },
            "Minimum": {
                "Measurement": 0
            },
            "Maximum": {
                "Measurement": 20
            },
            "Range": {
                "Measurement": 20
            },
            "Total": {
                "Measurement": 20
            },
            "Mean": {
                "Measurement": 10
            },
            "StandardDeviation": {
                "Measurement": 7.0710678118654755
            },
            "PopulationStandardDeviation": {
                "Measurement": 5
            },
            "WeightedMean": {
                "Measurement": 10
            },
            "WeightedStandardDeviation": {
                "Measurement": 7.0710678118654755
            },
            "WeightedPopulationStandardDeviation": {
                "Measurement": 5
            },
            "Skewness": {
                "Measurement": 0
            },
            "Kurtosis": {
                "Measurement": -2
            }
        }
    },
    {
        "Start": {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 20
        },
        "End": {
            "Time": "2017-11-23T16:00:00Z",
            "State": 0,
            "Measurement": 40
        },
        "Summaries": {
            "Count": {
                "Measurement": 2
            },
            "Minimum": {
                "Measurement": 20
            },
            "Maximum": {
                "Measurement": 40
            },
            "Range": {
                "Measurement": 20
            },
            "Total": {
                "Measurement": 60
            },
            "Mean": {
                "Measurement": 30
            },
            "StandardDeviation": {
                "Measurement": 7.0710678118654755
            },
            "PopulationStandardDeviation": {
                "Measurement": 5
            },
            "WeightedMean": {
                "Measurement": 30
            },
            "WeightedStandardDeviation": {
                "Measurement": 7.0710678118654755
            },
            "WeightedPopulationStandardDeviation": {
                "Measurement": 5
            },
            "Skewness": {
                "Measurement": 0
            },
            "Kurtosis": {
                "Measurement": -2
            }
        }
    }
]
```

*****

## ``Get Sampled Values``

Returns data sampled by intervals between a specified start and end index.

Sampling is driven by a specified property or properties of the stream's SdsType. Property types that cannot be interpolated do not support sampling requests. Strings are an example of a property that cannot be interpolated. For more information, see [Interpolation.](xref:ReadCharacteristics#interpolation)

**Request**  

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/{streamId}/Data/
    Sampled?startIndex={startIndex}&endIndex={endIndex}&intervals={intervals}&sampleBy={sampleBy}
    &boundaryType={boundaryType}&startBoundaryType={startBoundaryType}
    &endBoundaryType={endBoundaryType}&filter={filter}&streamViewId={streamViewId}
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streamId``  
The stream identifier.

``string startIndex``  
The start index for the intervals.

``string endIndex``  
The end index for the intervals.

``int intervals``  
The number of intervals requested.

``string sampleBy``  
Property or properties to use when sampling.

``SdsBoundaryType boundaryType``  
Optional SdsBoundaryType specifies the handling of events at or near the startIndex and endIndex.

``SdsBoundaryType startBoundaryType``  
Optional SdsBoundaryType specifies the handling of events at or near the startIndex.

``SdsBoundaryType endBoundaryType``  
Optional SdsBoundaryType specifies the handling of events at or near the endIndex.

``string filter``  
Optional filter expression (see [Filter expressions](xref:sdsFilterExpressions)).

**Response**  
The response includes a status code and a response body containing a serialized collection of events.

**Example**  
The following request returns two sample intervals between the `startIndex` and `endIndex`:

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Streams/Simple/Data/
    Sampled?startIndex=2019-01-01T00:00:00Z&endIndex=2019-01-02T00:00:00Z&intervals=2&sampleBy=Measurement
```

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2019-01-01T00:00:01Z",
        "State": 1,
        "Measurement": 1
    },
    {
        "Time": "2019-01-01T00:11:50Z",
        "State": 2,
        "Measurement": 0.00006028870675578446
    },
    {
        "Time": "2019-01-01T11:55:33Z",
        "Measurement": 6.277981349066863
    },
    {
        "Time": "2019-01-01T12:00:00Z",
        "Measurement": 3.101013140344655
    },
    {
        "Time": "2019-01-01T12:00:01Z",
        "State": 1,
        "Measurement": 4.101013140344655
    },
    {
        "Time": "2019-01-01T12:01:50Z",
        "State": 2,
        "Measurement": 0.0036776111121028521
    },
    {
        "Time": "2019-01-01T23:57:23Z",
        "State": 2,
        "Measurement": 6.2816589601789659
    },
    {
        "Time": "2019-01-02T00:00:00Z",
        "Measurement": 6.20202628068931
    }
]
```

**Note:** `State` is not included in the JSON when its value is the default value.
*****

## ``Join Values``

Returns data from multiple streams, which are joined based on the request specifications. The streams must be of the same SdsType.

SDS supports the following types of joins:

| SdsJoinMode  | Enumeration value | Operation |
| -------      | ----------------- | --------- |
| Inner        | 0                 | Results include the stored events with common indexes across specified streams. |
| Outer        | 1                 | Results include the stored events for all indexes across all streams. |
| Interpolated | 2                 | Results include events for each index across all streams for the request index boundaries. Some events may be interpolated. |
| MergeLeft    | 3                 | Results include events for each index across all streams selecting events at the indexes based on left to right order of the streams. |
| MergeRight   | 4                 | Results include events for each index across all streams selecting events at the indexes based on right to left order of the streams. |

SDS supports two types of join requests:

* [GET](#getjoin): The stream, joinMode, start index, and end index are specified in the request URI path.
* [POST](#postjoin): Only the SdsJoinMode is specified in the URI. The streams and read specification for each stream are specified in the body of the request.

<a name="getjoin"></a>

### `GET Request`

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?streams={streams}&joinMode={joinMode}
    &startIndex={startIndex}&endIndex={endIndex}
```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``string streams``  
Commas separated list of stream identifiers.

``SdsJoinMode joinMode``  
Type of join, that is inner, outer, and so on.

``string startIndex``  
Index identifying the beginning of the series of events to return.

``string endIndex``  
Index identifying the end of the series of events to return.

**Response**  
The response includes a status code and a response body containing multiple serialized events. See examples for specifics.

#### Examples

To join multiple streams, for example Simple1 and Simple2, assume that Simple1 presents the following data:

```json  
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T11:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 30
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "State": 0,
        "Measurement": 40
    }
]
```

And assume that Simple2 presents the following data:

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T12:00:00Z",
        "State": 0,
        "Measurement": 50
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 60
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "State": 0,
        "Measurement": 70
    },
    {
        "Time": "2017-11-23T17:00:00Z",
        "State": 0,
        "Measurement": 80
    }
]
```

The following are responses for various Joins request options:

##### Inner join example

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?streams=Simple1,Simple2&joinMode=inner
    &startIndex=0001-01-01T00:00:00.0000000&endIndex=9999-12-31T23:59:59.9999999
 ```

**Response**  
Measurements from both streams with common indexes.

**Response body** 

```json
HTTP/1.1 200
Content-Type: application/json

[
    [
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 30
        },
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 60
        }
    ]
]
```

##### Outer join example

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?streams=Simple1,Simple2&joinMode=outer
    &startIndex=0001-01-01T00:00:00.0000000&endIndex=9999-12-31T23:59:59.9999999
 ```

**Response**  
All Measurements from both Streams, with default values at indexes where a Stream does not have a value.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    [
        {
            "Time": "2017-11-23T11:00:00Z",
            "State": 0,
            "Measurement": 10
        },
        null
    ],
    [
        null,
        {
            "Time": "2017-11-23T12:00:00Z",
            "State": 0,
            "Measurement": 50
        }
    ],
    [
        {
            "Time": "2017-11-23T13:00:00Z",
            "State": 0,
            "Measurement": 20
        },
        null
    ],
    [
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 30
        },
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 60
        }
    ],
    [
        null,
        {
            "Time": "2017-11-23T15:00:00Z",
            "State": 0,
            "Measurement": 70
        }
    ],
    [
        {
            "Time": "2017-11-23T16:00:00Z",
            "State": 0,
            "Measurement": 40
        },
        null
    ],
    [
        null,
        {
            "Time": "2017-11-23T17:00:00Z",
            "State": 0,
            "Measurement": 80
        }
    ]
]
```

##### Interpolated join example

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?streams=Simple1,Simple2&joinMode=interpolated
    &startIndex=0001-01-01T00:00:00.0000000&endIndex=9999-12-31T23:59:59.9999999
 ```

**Response**  
All measurements from both Streams with missing values interpolated. If the missing values are between valid measurements within a stream, they are interpolated. If the missing values are outside of the boundary values, they are extrapolated.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    [
        {
            "Time": "2017-11-23T11:00:00Z",
            "State": 0,
            "Measurement": 10
        },
        {
            "Time": "2017-11-23T11:00:00Z",
            "State": 0,
            "Measurement": 50
        }
    ],
    [
        {
            "Time": "2017-11-23T12:00:00Z",
            "State": 0,
            "Measurement": 15
        },
        {
            "Time": "2017-11-23T12:00:00Z",
            "State": 0,
            "Measurement": 50
        }
    ],
    [
        {
            "Time": "2017-11-23T13:00:00Z",
            "State": 0,
            "Measurement": 20
        },
        {
            "Time": "2017-11-23T13:00:00Z",
            "State": 0,
            "Measurement": 55
        }
    ],
    [
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 30
        },
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 60
        }
    ],
    [
        {
            "Time": "2017-11-23T15:00:00Z",
            "State": 0,
            "Measurement": 35
        },
        {
            "Time": "2017-11-23T15:00:00Z",
            "State": 0,
            "Measurement": 70
        }
    ],
    [
        {
            "Time": "2017-11-23T16:00:00Z",
            "State": 0,
            "Measurement": 40
        },
        {
            "Time": "2017-11-23T16:00:00Z",
            "State": 0,
            "Measurement": 75
        }
    ],
    [
        {
            "Time": "2017-11-23T17:00:00Z",
            "State": 0,
            "Measurement": 40
        },
        {
            "Time": "2017-11-23T17:00:00Z",
            "State": 0,
            "Measurement": 80
        }
    ]
]
```

##### MergeLeft join example

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?streams=Simple1,Simple2&joinMode=mergeleft
    &startIndex=0001-01-01T00:00:00.0000000&endIndex=9999-12-31T23:59:59.9999999
 ```

**Response**  
This is similar to [OuterJoin](#outer-join-example), but the value at each index is the first available value at that index when iterating the given list of streams from left to right.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T11:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T12:00:00Z",
        "State": 0,
        "Measurement": 50
    },
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 30
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "State": 0,
        "Measurement": 70
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "State": 0,
        "Measurement": 40
    },
    {
        "Time": "2017-11-23T17:00:00Z",
        "State": 0,
        "Measurement": 80
    }
]
```

##### MergeRight join example

```text
GET api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?streams=Simple1,Simple2&joinMode=mergeright
    &startIndex=0001-01-01T00:00:00.0000000&endIndex=9999-12-31T23:59:59.9999999
 ```

**Response**  
This is similar to [OuterJoin](#outer-join-example), but the value at each index is the first available value at that index when iterating the given list of streams from right to left.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    {
        "Time": "2017-11-23T11:00:00Z",
        "State": 0,
        "Measurement": 10
    },
    {
        "Time": "2017-11-23T12:00:00Z",
        "State": 0,
        "Measurement": 50
    },
    {
        "Time": "2017-11-23T13:00:00Z",
        "State": 0,
        "Measurement": 20
    },
    {
        "Time": "2017-11-23T14:00:00Z",
        "State": 0,
        "Measurement": 60
    },
    {
        "Time": "2017-11-23T15:00:00Z",
        "State": 0,
        "Measurement": 70
    },
    {
        "Time": "2017-11-23T16:00:00Z",
        "State": 0,
        "Measurement": 40
    },
    {
        "Time": "2017-11-23T17:00:00Z",
        "State": 0,
        "Measurement": 80
    }
]
```

<a name="postjoin"></a>

### POST request

```text
POST api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
    ?joinMode={joinMode}
 ```

**Parameters**  
``string namespaceId``  
The namespace; either default or diagnostics.

``SdsJoinMode joinMode``  
Type of join, that is inner, outer, and so on.

**Request body**  
Read options specific to each stream.

**Response**  
The response includes a status code and a response body containing multiple serialized events.

Consider the following outer join request:

```text
    POST api/v1/Tenants/default/Namespaces/{namespaceId}/Bulk/Streams/Data/Joins
        ?joinMode=outer
 ```

where in the request body, different start indexes and end indexes are specified per stream:

```json
[  
    {  
        "StreamId": "Simple1",
        "Options":
        {
            "StartIndex": "2017-11-23T11:00:00Z",
            "EndIndex": "2017-11-23T14:00:00Z",
            "StartBoundaryType": "Exact",
            "EndBoundaryType": "Exact",
            "Count": 100,
            "Filter": ""
        }
    },
    {
        "StreamId": "Simple2",
        "Options":
        {
            "StartIndex": "2017-11-23T15:00:00Z",
            "EndIndex": "2017-11-23T17:00:00Z",
            "StartBoundaryType": "Exact",
            "EndBoundaryType": "Exact",
            "Count": 100,
            "Filter": ""
        }
    }
]
```

Only events within the stream's specified index boundaries are considered for the outer join operation.

**Response body**

```json
HTTP/1.1 200
Content-Type: application/json

[
    [
        {
            "Time": "2017-11-23T11:00:00Z",
            "State": 0,
            "Measurement": 10
        },
        null
    ],
    [
        {
            "Time": "2017-11-23T13:00:00Z",
            "State": 0,
            "Measurement": 20
        },
        null
    ],
    [
        {
            "Time": "2017-11-23T14:00:00Z",
            "State": 0,
            "Measurement": 30
        },
        null
    ],
    [
        null,
        {
            "Time": "2017-11-23T15:00:00Z",
            "State": 0,
            "Measurement": 70
        }
    ],
    [
        null,
        {
            "Time": "2017-11-23T17:00:00Z",
            "State": 0,
            "Measurement": 80
        }
    ]
]
```

**Note:** Not all the values from streams were included since they are restricted by individual queries for each Stream.
*****
  
