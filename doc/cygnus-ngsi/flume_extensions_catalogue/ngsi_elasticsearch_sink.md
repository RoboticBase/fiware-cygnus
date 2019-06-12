# <a name="top"></a>NGSIElasticsearchSink
Content:

* [Functionality](#section1)
    * [Mapping NGSI events to `NGSIEvent` objects](#section1.1)
    * [Mapping `NGSIEvent`s to Elasticsearch data structures](#section1.2)
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

#### <a name="section1.2.2"></a>Casting the value according to `type`
If `cast_value` parameter is set to `true`, the `attrValue` of `NGSIEvent` is cast according to the `attrType` when storing index.

|`attrType` (ignore case)|cast type|
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
The type of `attrValue` handled by Elasticsearch is determined by the first registered record. Therefore, when you set the `attr_persistence` parameter as `row` and `cast_value` parameter as `true`, 
