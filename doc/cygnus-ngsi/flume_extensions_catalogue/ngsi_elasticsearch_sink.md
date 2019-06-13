# <a name="top"></a>NGSIElasticsearchSink
Content:

* [Functionality](#section1)
    * [Mapping NGSI events to `NGSIEvent` objects](#section1.1)
    * [Mapping `NGSIEvent`s to Elasticsearch data structures](#section1.2)
        * [Elasticsearch index naming conventions](#section1.2.1)
        * [Converting the type of `attrValue` according to `attrType`](#section1.2.2)
        * [Row-like storing](#section1.2.3)
        * [Column-like storing](#section1.2.4)
    * [Example](#section1.3)
        * [`NGSIEvent`](#section1.3.1)
        * [Index names](#section1.3.2)
        * [Row-like storing](#section1.3.3)
        * [Column-like storing](#section1.3.4)
* [Administration guide](#section2)
* [Programmers guide](#section3)

## <a name="section1"></a>Functionality
`com.telefonica.iot.cygnus.sinks.NGSIElasticsearchSink`, or simply `NGSIElasticsearchSink` is a sink designed to persist NGSI context data events into an Elasticsearch. Usually, such a NGSI context data is notified from a [Orion Context Broker](https://github.com/telefonicaid/fiware-orion), but any other sources can be accepted as long as they are NGSI.

Independently of the data generator, NGSI context data is always transformed into internal `NGSIEvent` objects at Cygnus sources. In the end, the information within these events must be mapped into specific Elasticsearch data structures.

Next sections will explain this is detail.

[Top](#top)

### <a name="section1.1"></a>Mapping NGSI events to `NGSIEvent` objects
Notified NGSI events (containing context data) are transformed into `NGSIEvent` objects (for each context element a `NGSIEvent` is created; such an event is a mix of certain headers and a `ContextElement` object), independently of the NGSI data generator or the final backend where it is persisted.

This is done at the cygnus-ngsi Http listeners (in Flume jergon, sources) thanks to [`NGSIRestHandler`](/ngsi_rest_handler.md). Once translated, the data (now, as `NGSIEvent` objects) is put into the internal channels for future consumption (see next section).

[Top](#top)

### <a name="section1.2"></a>Mapping `NGSIEvent`s to Elasticsearch data structures
Elasticsearch organizes the data in database that contain collections of Json documents. Such organization is exploited by `NGSIElasticsearchSink` each time a `NGSIEvent` is going to be persisted.

#### <a name="section1.2.1"></a>Elasticsearch index naming conventions
An index of Elasticsearch called as the `fiware-service` header value within the event is created (if not existing yet). A configured prefix is added (by default, `cygnus`).

The Elasticsearch index name has some limitations such as [the index name is lowercase only , it cannot include `\, /, *, ?, ", <, >, |, ` ` (space character), ,, #, :` and it cannot start with `-, _, +`](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/indices-create-index.html). So NGSIElasticsearchSink constructs the index name according to the following steps:
1. join prefix, "-", fiware service  and fiware servicepath.
2. convert to lower cases.
3. replace forbidden charactters to '-'.
4. append 'idx' at the beggning of index name when it starts with `-, _, +`.

The Elasticsearch index is limited to 255 bytes.

[Top](#top)

#### <a name="section1.2.2"></a>Converting the type of `attrValue` according to `attrType`
If `cast_value` parameter is set to `true`, the type of `attrValue` will be converted automatically according to the `attrType` when storing index. The converting rule is like below:

|`attrType` (ignore case)|the type to be converted|
|:--|:--|
|`int` or `integer`|Integer|
|`float`|Float|
|`number` or `double`|Double|
|`bool` or `boolean`|Boolean|
|otherwise|String|

[Top](#top)

#### <a name="section1.2.3"></a>Row-like storing
Regarding the specific data stored within the above index, if `attr_persistence` parameter is set to `row` (default storing mode) then the notified data is stored attribute by attribute, composing a Json document for each one of them. Each document contains the following fields:

* `recvTime`: timestamp in human-readable format ([ISO 8601](http://en.wikipedia.org/wiki/ISO_8601)). You can set the timezone of recvTime by using the `timezone` parameter.
* `entityId`: Notified entity identifier.
* `entityType`: Notified entity type.
* `attrName`: Notified attribute name.
* `attrType`: Notified attribute type.
* `attrValue`: Notified atribute value. If `cast_value` parameter is set to `true`, this value is automatically cast. Otherwise the value is treated as String.
* `attrMetadata`: Notified attribute metadata.

**Caution**
The type of `attrValue` handled by Elasticsearch is determined by the first registered record. Therefore, when you set the `attr_persistence` parameter as `row` and `cast_value` parameter as `true`, the later attribute records which have different type with the first attribute record will be ignored and will not be stored to Elasticsearch.

[Top](#top)

#### <a name="section1.2.4"></a>Column-like storing
Regarding the specific data stored within the above collections, if `attr_persistence` parameter is set to `column` then a single Json document is composed for the whole notified entity. Each document contains a variable number of fields:

* `recvTime`: timestamp in human-readable format ([ISO 8601](http://en.wikipedia.org/wiki/ISO_8601)). You can set the timezone of recvTime by using the `timezone` parameter.
* `entityId`: Notified entity identifier.
* `entityType`: Notified entity type.
*  For each notified attribute, a field named as the attribute is considered. This field will store the attribute values along the time.

**Caution**
When `attr_persistence` parameter is set to `column`, the metadata of each attribute will be ignored.

### <a name="section1.3"></a>Example
#### <a name="section1.3.1"></a>`NGSIEvent`
Assuming the following `NGSIEvent` is created from a notified NGSI context data (the code below is an <i>object representation</i>, not any real data format):

    ngsi-event={
        headers={
	         content-type=application/json,
	         timestamp=1429535775,
	         transactionId=1429535775-308-0000000000,
	         correlationId=1429535775-308-0000000000,
	         fiware-service=vehicles,
	         fiware-servicepath=/4wheels,
	         <grouping_rules_interceptor_headers>,
	         <name_mappings_interceptor_headers>
        },
        body={
	        entityId=car1,
	        entityType=car,
	        attributes=[
	            {
	                attrName=speed,
	                attrType=float,
	                attrValue=112.9
	            },
	            {
	                attrName=oil_level,
	                attrType=float,
	                attrValue=74.6
	            },
	            {
	                attrName=driver,
	                attrType=string,
	                attrValue=Jhon
	            },
	            {
	                attrName=headlight,
	                attrType=boolean,
	                attrValue=true
	            }
	        ]
	    }
    }

[Top](#top)

#### <a name="section1.3.2"></a>Index names
A Elasticsearch index is named as the concatenation of prefix , "-", the notified FIWARE servcie, FIWARE service path and created date (yyyy.mm.dd). The default value of prefix is `cygnus`, but you can change it by using the `index_prefix` parameter.
The concatinated string will be lower cased, and some forbidden characters(`\, /, *, ?, ", <, >, |, ` ` (space character), ,, #, :`) will be replaced by '-'. And then, 'idx' will be appended at the beggning when prefix starts with `-, _, +`.

|`date`|`prefix`|`FIWARE service`|`FIWARE service path`|`index name`|
|:--|:--|:--|:--|:--|
|June 13, 2019|`_#PREFIX*1`|`vehicles`|`/4wheels`|`idx_-prefix-1-vehicles-4wheels-2019.06.13`|

[Top](#top)

#### <a name="section1.3.3"></a>Row-like storing
Assuming `attr_persistence=row` and `cast_value=false` as configuration parameters, then `NGSIElasticsearchSink` will persist the 4 records within its index as:

    {
      "_index": "idx_-prefix-1-vehicles-4wheels-2019.06.13",
      "_type": "cygnus_type",
      "_id": "1560410492045-4B16059D92EF1CF00BC343462A5809FE",
      "_version": 1,
      "_score": null,
      "_source": {
        "recvTime": "2019-06-13T16:21:32.045+0900",
        "entityType": "car",
        "attrMetadata": [],
        "entityId": "car1",
        "attrValue": "Jhon",
        "attrName": "driver",
        "attrType": "string"
      },
      "fields": {
        "recvTime": [
          "2019-06-13T07:21:32.045Z"
        ]
      },
      "sort": [
        1560410492045
      ]
    }

    {
      "_index": "idx_-prefix-1-vehicles-4wheels-2019.06.13",
      "_type": "cygnus_type",
      "_id": "1560410492045-CDCE5B76B4E507455E43748C66A6544E",
      "_version": 1,
      "_score": null,
      "_source": {
        "recvTime": "2019-06-13T16:21:32.045+0900",
        "entityType": "car",
        "attrMetadata": [],
        "entityId": "car1",
        "attrValue": "true",
        "attrName": "headlight",
        "attrType": "boolean"
      },
      "fields": {
        "recvTime": [
          "2019-06-13T07:21:32.045Z"
        ]
      },
      "sort": [
        1560410492045
      ]
    }

    {
      "_index": "idx_-prefix-1-vehicles-4wheels-2019.06.13",
      "_type": "cygnus_type",
      "_id": "1560410492045-1542A836CA541586B42516D054EBD187",
      "_version": 1,
      "_score": null,
      "_source": {
        "recvTime": "2019-06-13T16:21:32.045+0900",
        "entityType": "car",
        "attrMetadata": [],
        "entityId": "car1",
        "attrValue": "74.6",
        "attrName": "oil_level",
        "attrType": "float"
      },
      "fields": {
        "recvTime": [
          "2019-06-13T07:21:32.045Z"
        ]
      },
      "sort": [
        1560410492045
      ]
    }

    {
      "_index": "idx_-prefix-1-vehicles-4wheels-2019.06.13",
      "_type": "cygnus_type",
      "_id": "1560410492045-D6CDC64848D72A71AE385CCD16D71CEA",
      "_version": 1,
      "_score": null,
      "_source": {
        "recvTime": "2019-06-13T16:21:32.045+0900",
        "entityType": "car",
        "attrMetadata": [],
        "entityId": "car1",
        "attrValue": "112.9",
        "attrName": "speed",
        "attrType": "float"
      },
      "fields": {
        "recvTime": [
          "2019-06-13T07:21:32.045Z"
        ]
      },
      "sort": [
        1560410492045
      ]
    }

[Top](#top)

#### <a name="section1.3.4"></a>Column-like storing
Assuming `attr_persistence=column` and `cast_value=true` as configuration parameters, then `NGSIElasticsearchSink` will persist a record within its index as:

    {
      "_index": "idx_-prefix-1-vehicles-4wheels-2019.06.13",
      "_type": "cygnus_type",
      "_id": "1560406984429-A9F56B9055EC751FD7E5941C43C90F29",
      "_version": 1,
      "_score": null,
      "_source": {
        "oil_level": 74.6,
        "recvTime": "2019-06-13T15:23:04.429+0900",
        "driver": "Jhon",
        "entityType": "car",
        "entityId": "car1",
        "speed": 112.9,
        "headlight": true
      },
      "fields": {
        "recvTime": [
          "2019-06-13T06:23:04.429Z"
        ]
      },
      "sort": [
        1560406984429
      ]
    }

Because `cast_value` parameter is set as true, Elasticsearch handles the `speed` and `oil_level` as **number**, `driver` as **string**, and `headlight` as **boolean**.

[Top](#top)

## <a name="section2"></a>Administration guide
### <a name="section2.1"></a>Configuration
`NGSIElasticsearchSink` is configured through the following parameters:

| Parameter | Mandatory | Default value | Comments |
|---|---|---|---|
| type | yes | N/A | com.telefonica.iot.cygnus.sinks.NGSIElasticsearchSink |
| channel | yes | N/A | elasticsearch-channel |
| elasticsearch\_host | yes | localhost | the hostname of Elasticsearch server |
| elasticsearch\_port | yes | 9200 | the port number of Elasticsearch server (0 - 65535) |
| ssl | yes | false | true if connect to Elasticsearch server using SSL ("true" or "false") |
| index\_prefix | no | cygnus | the prefix of index name |
| mapping\_type | no | cygnus\_type | the mapping type name of Elasticsearch |
| ignore\_white\_spaces | no | true | true if exclusively white space-based attribute values must be ignored, false otherwise ("true" or "false") |
| attr\_persistence | no | row | the persistence style as row-style or column-style ("row" or "column") |
| timezone | no | UTC | timezone to be used as a document's timestamp |
| cast\_value | no | false | true if cast the attrValue using attrType ("true" or "false") |
| cache\_flash\_interval\_sec | no | 0 | 0 if notified data will be persisted to Elasticsearch immediately. positive integer if notified data are cached on NGSIElasticsearchSink's memory and will be persisted to Elasticsearch periodically every `cache_flash_interval_sec` |
| backend.max\_conns | no | 500 | Maximum number of connections allowed for a Http-based HDFS backend |
| backend.max\_conns\_per\_route | no | 100 | Maximum number of connections per route allowed for a Http-based HDFS backend |

A configuration example could be:

    cygnus-ngsi.sinks = elasticsearch-sink
    cygnus-ngsi.channels = elasticsearch-channel
    ...
    cygnus-ngsi.sinks.elasticsearch-sink.type = com.telefonica.iot.cygnus.sinks.NGSIElasticsearchSink
    cygnus-ngsi.sinks.elasticsearch-sink.channel = elasticsearch-channel
    cygnus-ngsi.sinks.elasticsearch-sink.elasticsearch_host = elasticsearch.local
    cygnus-ngsi.sinks.elasticsearch-sink.elasticsearch_port = 9200
    cygnus-ngsi.sinks.elasticsearch-sink.ssl = false
    cygnus-ngsi.sinks.elasticsearch-sink.index_prefix = cygnus
    cygnus-ngsi.sinks.elasticsearch-sink.mapping_type = cygnus_type
    cygnus-ngsi.sinks.elasticsearch-sink.ignore_white_spaces = true
    cygnus-ngsi.sinks.elasticsearch-sink.attr_persistence = row
    cygnus-ngsi.sinks.elasticsearch-sink.timezone = UTC
    cygnus-ngsi.sinks.elasticsearch-sink.cast_value = false
    cygnus-ngsi.sinks.elasticsearch-sink.cache_flash_interval_sec = 0
    cygnus-ngsi.sinks.elasticsearch-sink.backend.max_conns = 500
    cygnus-ngsi.sinks.elasticsearch-sink.backend.max_conns_per_route = 100

[Top](#top)

### <a neme="section2.2"></a>Use cases
Use `NGSIElasticsearchSink` if you are looking for a Json-based full-text search engine.

[Top](#top)

### <a name="section2.3"></a>Important notes
#### <a name="section2.3.1"></a>About batching
