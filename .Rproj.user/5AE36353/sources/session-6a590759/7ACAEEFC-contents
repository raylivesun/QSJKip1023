KIP-1033: Add Kafka Streams exception handler for exceptions occurring
during processing
================

Created by [Damien
Gasparina](https://cwiki.apache.org/confluence/display/~d.gasparina),
last modified [yesterday at 07:20
AM](https://cwiki.apache.org/confluence/pages/diffpagesbyversion.action?pageId=300026309&selectedPageVersions=41&selectedPageVersions=42 "Show changes")

- [Status](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-Status)

- [Motivation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-Motivation)

- [Proposed
  Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-ProposedChanges)

  - [Metrics](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-Metrics)

- [Public
  Interfaces](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-PublicInterfaces)

- [Compatibility, Deprecation, and Migration
  Plan](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-Compatibility,Deprecation,andMigrationPlan)

- [Examples](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-Examples)

- [Test
  Plan](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-TestPlan)

- [Rejected
  Alternatives](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1033%3A+Add+Kafka+Streams+exception+handler+for+exceptions+occurring+during+processing#KIP1033:AddKafkaStreamsexceptionhandlerforexceptionsoccurringduringprocessing-RejectedAlternatives)

# Status

**Current state**: *Accepted (targeting 3.8)  
*

**Discussion thread**:
[here](https://lists.apache.org/thread/1nhhsrogmmv15o7mk9nj4kvkb5k2bx9s)*  
*

**JIRA**: [here](https://issues.apache.org/jira/browse/KAFKA-16448)

Please keep the discussion on the mailing list rather than commenting on
the wiki (wiki discussions get unwieldy fast).

# Motivation

*KIP inspired by Michelin
[kstreamplify](https://github.com/michelin/kstreamplify) and coauthored
by Damien Gasparina, Loïc Greffier and Sebastien Viale.*

There are several places where an exception could arise in Kafka
Streams. Currently, developers can implement a
`DeserializationExceptionHandler`, to handle issues during the
deserialization, and a `ProductionExceptionHandler` for issues happening
during the production. 

For issues happening during the process of a message, it’s up to the
developer to add the required try/catch when they leverage the DSL or
the Processor API. All uncaught exceptions will terminate the processing
or the StreamThread. This approach is quite tedious and error prone when
you are developing a large topology.

This proposal aims to add a new Exception handling mechanism to manage
exceptions happening during the processing of a message. It will be
invoked if an exception happens during the invocation of a
Processor.process. Similar to the other exception handler, it would
include two out of the box implementations: `LogAndFail`, the default to
be backward compatible, and `LogAndContinue`.

This KIP also proposes to unify the method signatures of the
DeserializationExceptionHandler and the ProductionExceptionHandler.
Currently, the DeserializationExceptionHandler is leveraging the old
ProcessorContext and the ProductionExceptionHandler does not have access
to any context. As the ProcessorContext is exposing many methods that
should not be accessed during the exception handler, e.g. forward(),
this KIP introduces a new container class exposing only the metadata of
the processing context: ErrorHandlerContext.

# Proposed Changes

We propose to add a new exception handler that could be set by the user
to handle exceptions happening during the process of a message.

The interface would be:

**ProcessingExceptionHandler.java**

<table>
<colgroup>
<col style="width: 4%" />
<col style="width: 95%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>1</p>
<p>2</p>
<p>3</p>
<p>4</p>
<p>5</p>
<p>6</p>
<p>7</p>
<p>8</p>
<p>9</p>
<p>10</p>
<p>11</p>
<p>12</p>
<p>13</p>
<p>14</p>
<p>15</p>
<p>16</p>
<p>17</p>
<p>18</p>
<p>19</p>
<p>20</p>
<p>21</p>
<p>22</p>
<p>23</p></td>
<td><p><code>package</code>
<code>org.apache.kafka.streams.errors;</code></p>
<p><code>/**</code></p>
<p><code>* An interface that allows user code to inspect a record that has failed processing</code></p>
<p><code>*/</code></p>
<p><code>public</code> <code>interface</code>
<code>ProcessingExceptionHandler extends</code>
<code>Configurable {</code></p>
<p><code>/**</code></p>
<p><code>* Inspect a record and the exception received</code></p>
<p><code>* @param context processing context metadata</code></p>
<p><code>* @param record record where the exception occurred</code></p>
<p><code>* @param exception the actual exception</code></p>
<p><code>*/</code></p>
<p><code>ProcessingHandlerResponse handle(ErrorHandlerContext context, Record&lt;?, ?&gt; record, Exception exception);</code></p>
<p><code>public</code> <code>enum</code>
<code>ProcessingHandlerResponse {</code></p>
<p><code>/* continue with processing */</code></p>
<p><code>CONTINUE(1, "CONTINUE"),</code></p>
<p><code>/* fail the processing and stop */</code></p>
<p><code>FAIL(2, "FAIL");</code></p>
<p><code>...</code></p>
<p><code>}</code></p>
<p><code>}</code></p></td>
</tr>
</tbody>
</table>

To not expose sensitive information during the handler invocation, this
KIP also introduces a new container class exposing only Processing
metadata to the handler, as those metadata could hold longer than the
ProcessingContext due the ProductionExceptionHandler:

**ErrorHandlerContext.java**

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><code>/**</code></p>
<p><code>* ErrorHandlerContext interface</code></p>
<p><code>*/</code></p>
<p><code>public</code> <code>interface</code>
<code>ErrorHandlerContext {</code></p>
<p><code>/**</code></p>
<p><code>* Return the topic name of the current input record; could be {@code null} if it is not</code></p>
<p><code>* available.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; For example, if this method is invoked within a {@link Punctuator#punctuate(long)</code></p>
<p><code>* punctuation callback}, or while processing a record that was forwarded by a punctuation</code></p>
<p><code>* callback, the record won't have an associated topic.</code></p>
<p><code>* Another example is</code></p>
<p><code>* {@link org.apache.kafka.streams.kstream.KTable#transformValues(ValueTransformerWithKeySupplier, String...)}</code></p>
<p><code>* (and siblings), that do not always guarantee to provide a valid topic name, as they might be</code></p>
<p><code>* executed "out-of-band" due to some internal optimizations applied by the Kafka Streams DSL.</code></p>
<p><code>*</code></p>
<p><code>* @return the topic name</code></p>
<p><code>*/</code></p>
<p><code>String topic();</code></p>
<p><code>/**</code></p>
<p><code>* Return the partition id of the current input record; could be {@code -1} if it is not</code></p>
<p><code>* available.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; For example, if this method is invoked within a {@link Punctuator#punctuate(long)</code></p>
<p><code>* punctuation callback}, or while processing a record that was forwarded by a punctuation</code></p>
<p><code>* callback, the record won't have an associated partition id.</code></p>
<p><code>* Another example is</code></p>
<p><code>* {@link org.apache.kafka.streams.kstream.KTable#transformValues(ValueTransformerWithKeySupplier, String...)}</code></p>
<p><code>* (and siblings), that do not always guarantee to provide a valid partition id, as they might be</code></p>
<p><code>* executed "out-of-band" due to some internal optimizations applied by the Kafka Streams DSL.</code></p>
<p><code>*</code></p>
<p><code>* @return the partition id</code></p>
<p><code>*/</code></p>
<p><code>int</code> <code>partition();</code></p>
<p><code>/**</code></p>
<p><code>* Return the offset of the current input record; could be {@code -1} if it is not</code></p>
<p><code>* available.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; For example, if this method is invoked within a {@link Punctuator#punctuate(long)</code></p>
<p><code>* punctuation callback}, or while processing a record that was forwarded by a punctuation</code></p>
<p><code>* callback, the record won't have an associated offset.</code></p>
<p><code>* Another example is</code></p>
<p><code>* {@link org.apache.kafka.streams.kstream.KTable#transformValues(ValueTransformerWithKeySupplier, String...)}</code></p>
<p><code>* (and siblings), that do not always guarantee to provide a valid offset, as they might be</code></p>
<p><code>* executed "out-of-band" due to some internal optimizations applied by the Kafka Streams DSL.</code></p>
<p><code>*</code></p>
<p><code>* @return the offset</code></p>
<p><code>*/</code></p>
<p><code>long</code> <code>offset();</code></p>
<p><code>/**</code></p>
<p><code>* Return the headers of the current source record; could be an empty header if it is not</code></p>
<p><code>* available.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; For example, if this method is invoked within a {@link Punctuator#punctuate(long)</code></p>
<p><code>* punctuation callback}, or while processing a record that was forwarded by a punctuation</code></p>
<p><code>* callback, the record might not have any associated headers.</code></p>
<p><code>* Another example is</code></p>
<p><code>* {@link org.apache.kafka.streams.kstream.KTable#transformValues(ValueTransformerWithKeySupplier, String...)}</code></p>
<p><code>* (and siblings), that do not always guarantee to provide valid headers, as they might be</code></p>
<p><code>* executed "out-of-band" due to some internal optimizations applied by the Kafka Streams DSL.</code></p>
<p><code>*</code></p>
<p><code>* @return the headers</code></p>
<p><code>*/</code></p>
<p><code>Headers headers();</code></p>
<p><code>/**</code></p>
<p><code>* Return the non-deserialized byte[] of the input message key if the context has been triggered by a message.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; If this method is invoked within a {@link Punctuator#punctuate(long)</code></p>
<p><code>* punctuation callback}, or while processing a record that was forwarded by a punctuation</code></p>
<p><code>* callback, it will return null.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; If this method is invoked in a sub-topology due to a repartition, the returned key would be one sent</code></p>
<p><code>* to the repartition topic.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; Always returns null if this method is invoked within a</code></p>
<p><code>* {@link ProductionExceptionHandler.handle(ErrorHandlerContext, ProducerRecord, Exception)}</code></p>
<p><code>*</code></p>
<p><code>* @return the raw byte of the key of the source message</code></p>
<p><code>*/</code></p>
<p><code>byte[] sourceRawKey();</code></p>
<p><code>/**</code></p>
<p><code>* Return the non-deserialized byte[] of the input message value if the context has been triggered by a message.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; If this method is invoked within a {@link Punctuator#punctuate(long)</code></p>
<p><code>* punctuation callback}, or while processing a record that was forwarded by a punctuation</code></p>
<p><code>* callback, it will return null.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; If this method is invoked in a sub-topology due to a repartition, the returned value would be one sent</code></p>
<p><code>* to the repartition topic.</code></p>
<p><code>*</code></p>
<p><code>* &lt;p&gt; Always returns null if this method is invoked within a</code></p>
<p><code>* {@link ProductionExceptionHandler.handle(ErrorHandlerContext, ProducerRecord, Exception)}</code></p>
<p><code>*</code></p>
<p><code>* @return the raw byte of the value of the source message</code></p>
<p><code>*/</code></p>
<p><code>byte[] sourceRawValue();</code></p>
<p><code>/**</code></p>
<p><code>* Return the current processor node id.</code></p>
<p><code>*</code></p>
<p><code>* @return the processor node id</code></p>
<p><code>*/</code></p>
<p><code>String processorNodeId();</code></p>
<p><code>/**</code></p>
<p><code>* Return the task id.</code></p>
<p><code>*</code></p>
<p><code>* @return the task id</code></p>
<p><code>*/</code></p>
<p><code>TaskId taskId();</code></p>
<p><code>}</code></p></td>
</tr>
</tbody>
</table>

To be consistent with other handlers, this KIP proposes to expose the
new ErrorHandlerContext as a parameter to the Deserialization and
Production exception handlers and deprecate the previous handle
signature. To be backward compatible with existing handlers, a default
implementation would be provided to ensure that the previous handler is
invoked by default. A default implementation for the deprecated method
is also implemented to allow user to not implement the deprecated method
while implementing the interface.

Note: to avoid memory pressure, the sourceRawKey and sourceRawValue
attributes would always be null during the
ProductionExceptionHandlerResponse.handle method.

The ProductionExceptionHandler interface will be:

**ProductionExceptionHandler.java**

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><code>/**</code></p>
<p><code>* ...</code></p>
<p><code>*/</code></p>
<p><code>public</code> <code>interface</code>
<code>ProductionExceptionHandler extends</code>
<code>Configurable {</code></p>
<p><code>/**</code></p>
<p><code>* ...</code></p>
<p><code>* @deprecated Please use the ProductionExceptionHandlerResponse.handle(metadata, record, exception)</code></p>
<p><code>*/</code></p>
<p><code>@Deprecated</code></p>
<p><code>default</code>
<code>ProductionExceptionHandlerResponse handle(final</code>
<code>ProducerRecord&lt;byte[], byte[]&gt; record,</code></p>
<p><code>final</code> <code>Exception exception) {</code></p>
<p><code>throw</code> <code>new</code>
<code>NotImplementedException();</code></p>
<p><code>}</code></p>
<p><code>/**</code></p>
<p><code>* Inspect a record that we attempted to produce, and the exception that resulted</code></p>
<p><code>* from attempting to produce it and determine whether or not to continue processing.</code></p>
<p><code>*</code></p>
<p><code>* @param context The error handler context metadata</code></p>
<p><code>* @param record The record that failed to produce</code></p>
<p><code>* @param exception The exception that occurred during production</code></p>
<p><code>*/</code></p>
<p><code>@SuppressWarnings("deprecation")</code></p>
<p><code>default</code>
<code>ProductionExceptionHandlerResponse handle(final</code>
<code>ErrorHandlerContext context,</code></p>
<p><code>final</code>
<code>ProducerRecord&lt;byte[], byte[]&gt; record,</code></p>
<p><code>final</code> <code>Exception exception) {</code></p>
<p><code>return</code> <code>handle(record, exception);</code></p>
<p><code>}</code></p>
<p><code>/**</code></p>
<p><code>* ...</code></p>
<p><code>* @deprecated          Please use the handleSerializationException(record, exception, context)</code></p>
<p><code>*/</code></p>
<p><code>@Deprecated</code></p>
<p><code>default</code>
<code>ProductionExceptionHandlerResponse handleSerializationException(final</code>
<code>ProducerRecord record,</code></p>
<p><code>final</code> <code>Exception exception) {</code></p>
<p><code>return</code>
<code>ProductionExceptionHandlerResponse.FAIL;</code></p>
<p><code>}</code></p>
<p><code>/**</code></p>
<p><code>* Handles serialization exception and determine if the process should continue. The default implementation is to</code></p>
<p><code>* fail the process.</code></p>
<p><code>*</code></p>
<p><code>* @param context       the error handler context metadata</code></p>
<p><code>* @param record        the record that failed to serialize</code></p>
<p><code>* @param exception     the exception that occurred during serialization</code></p>
<p><code>*/</code></p>
<p><code>@SuppressWarnings("deprecation")</code></p>
<p><code>default</code>
<code>ProductionExceptionHandlerResponse handleSerializationException(final</code>
<code>ErrorHandlerContext context,</code></p>
<p><code>final</code> <code>ProducerRecord record,</code></p>
<p><code>final</code> <code>Exception exception,</code></p>
<p><code>final</code>
<code>SerializationExceptionOrigin origin) {</code></p>
<p><code>return</code>
<code>handleSerializationException(record, exception);</code></p>
<p><code>}</code></p>
<p><code>}</code></p></td>
</tr>
</tbody>
</table>

The deserializationExceptionHandler would be:

**DeserializationExceptionHandler.java**

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><code>public</code> <code>interface</code>
<code>DeserializationExceptionHandler extends</code>
<code>Configurable {</code></p>
<p><code>// ...</code></p>
<p><code>/*</code></p>
<p><code>* ...</code></p>
<p><code>* @deprecated Please use the DeserializationExceptionHandlerResponse.handle(errorContextMetadata, record, errorHandlerContext)</code></p>
<p><code>*/</code></p>
<p><code>@Deprecated</code></p>
<p><code>default</code>
<code>DeserializationHandlerResponse handle(final</code>
<code>ProcessorContext context,</code></p>
<p><code>final</code>
<code>ConsumerRecord&lt;byte[], byte[]&gt; record,</code></p>
<p><code>final</code> <code>Exception exception) {</code></p>
<p><code>throw</code> <code>new</code>
<code>NotImplementedException();</code></p>
<p><code>}</code></p>
<p><code>/**</code></p>
<p><code>* Inspect a record and the exception received.</code></p>
<p><code>*</code></p>
<p><code>* @param context error handler context</code></p>
<p><code>* @param record record that failed deserialization</code></p>
<p><code>* @param exception the actual exception</code></p>
<p><code>*/</code></p>
<p><code>default</code>
<code>DeserializationHandlerResponse handle(final</code>
<code>ErrorHandlerContext context,</code></p>
<p><code>final</code>
<code>ConsumerRecord&lt;byte[], byte[]&gt; record,</code></p>
<p><code>final</code> <code>Exception exception) {</code></p>
<p><code>return</code>
<code>handle(((ErrorHandlerContextImpl) context).convertToProcessorContext(), record, exception);</code></p>
<p><code>}</code></p>
<p><code>// ...</code></p>
<p><code>}</code></p></td>
</tr>
</tbody>
</table>

To identify the origin of the serialization exception, a
new SerializationExceptionOrigin would be provided in the serialization
handler:

**SerializationExceptionOrigin.java**

<table style="width:65%;">
<colgroup>
<col style="width: 65%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><code>package</code>
<code>org.apache.kafka.streams.errors;</code></p>
<p><code>enum</code> <code>SerializationExceptionOrigin {</code></p>
<p><code>KEY,</code></p>
<p><code>VALUE</code></p>
<p><code>}</code></p></td>
</tr>
</tbody>
</table>

## Metrics

The following metrics will be incremented each time the
ProcessingExceptionHandler has been invoked and returned CONTINUE.  

<table style="width:99%;">
<colgroup>
<col style="width: 9%" />
<col style="width: 14%" />
<col style="width: 16%" />
<col style="width: 21%" />
<col style="width: 36%" />
</colgroup>
<tbody>
<tr class="odd">
<td></td>
<td>LEVEL 0</td>
<td>LEVEL 1</td>
<td>LEVEL 2</td>
<td>LEVEL 3</td>
</tr>
<tr class="even">
<td></td>
<td>Per-Client</td>
<td>Per-Thread</td>
<td>Per-Task</td>
<td>Per-Processor-Node</td>
</tr>
<tr class="odd">
<td>TAGS</td>
<td><em>type=stream-metrics,client-id=[client-id]</em></td>
<td><em>type=stream-thread-metrics,thread-id=[threadId]</em></td>
<td><em>type=stream-task-metrics,thread-id=[threadId],task-id=[taskId]</em></td>
<td><em>type=stream-processor-node-metrics,thread-id=[threadId],task-id=[taskId],processor-node-id=[processorNodeId]</em></td>
</tr>
<tr class="even">
<td><pre><code>dropped-records (rate|total)</code></pre></td>
<td></td>
<td></td>
<td>INFO</td>
<td></td>
</tr>
</tbody>
</table>

# Public Interfaces

Users will be able to set an exception handler through the following new
config option which points to a class name.

`public static final String PROCESSING_EXCEPTION_HANDLER_CLASS_CONFIG = "processing.exception.handler".`

Two implementations of the interface will be provided. The default will
be the `LogAndFailExceptionHandler` to ensure the backward compatibility
of the API

<table>
<colgroup>
<col style="width: 4%" />
<col style="width: 95%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>1</p>
<p>2</p>
<p>3</p>
<p>4</p>
<p>5</p>
<p>6</p>
<p>7</p>
<p>8</p>
<p>9</p>
<p>10</p>
<p>11</p>
<p>12</p></td>
<td><p><code>// logs the error and returns CONTINUE</code></p>
<p><code>public</code> <code>class</code>
<code>ProcessingLogAndContinueExceptionHandler implements</code>
<code>ProcessingExceptionHandler {...}</code></p>
<p><code>// logs the error and returns FAIL</code></p>
<p><code>public</code> <code>class</code>
<code>ProcessingLogAndFailExceptionHandler implements</code>
<code>ProcessingExceptionHandler {...}</code></p>
<p><code>// Then in StreamsConfig.java:</code></p>
<p><code>.define(PROCESSING_EXCEPTION_HANDLER_CLASS_CONFIG,</code></p>
<p><code>Type.CLASS,</code></p>
<p><code>ProcessingLogAndFailExceptionHandler.class.getName(),</code></p>
<p><code>Importance.MEDIUM,</code></p>
<p><code>PROCESSING_EXCEPTION_HANDLER_CLASS_DOC)</code></p></td>
</tr>
</tbody>
</table>

# Compatibility, Deprecation, and Migration Plan

In the current version, exceptions thrown during the processing of a
message are not caught, and thus will be sent to the
`uncaughtExceptionHandler` and will terminate the StreamThread. To be
backward compatible, the `ProcessingLogAndFailExceptionHandler` will be
set as default.

# Examples

**LogAndContinueProcessingExceptionHandler.java**

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><code>public</code> <code>class</code>
<code>LogAndContinueProcessingExceptionHandler implements</code>
<code>ProcessingExceptionHandler {</code></p>
<p><code>private</code> <code>static</code> <code>final</code>
<code>Logger log = LoggerFactory.getLogger(LogAndContinueProcessingExceptionHandler.class);</code></p>
<p><code>@Override</code></p>
<p><code>public</code> <code>void</code>
<code>configure(Map&lt;String, ?&gt; configs) {</code></p>
<p><code>// ignore</code></p>
<p><code>}</code></p>
<p><code>@Override</code></p>
<p><code>public</code>
<code>ProcessingHandlerResponse handle(ErrorHandlerContext context, Record&lt;?, ?&gt; record, Exception exception) {</code></p>
<p><code>log.warn("Exception caught during message processing, "</code>
<code>+</code></p>
<p><code>"processor node: {}, taskId: {}, source topic: {}, source partition: {}, source offset: {}",</code></p>
<p><code>context.processorNodeId(), context.taskId(), context.topic(), context.partition(), context.offset(),</code></p>
<p><code>exception);</code></p>
<p><code>return</code>
<code>ProcessingHandlerResponse.CONTINUE;</code></p>
<p><code>}</code></p>
<p><code>}</code></p></td>
</tr>
</tbody>
</table>

# Test Plan

- A unit test will be implemented to ensure that exceptions are caught
  by the `ProcessingExceptionHandler`and the provided implementations.

- A unit test will be implemented to ensure the backward compatibility.

- An integration test will be done to ensure that the right exceptions
  are caught.

# Rejected Alternatives

- **Allow per node exception handler**, specifying an exception handler
  could be error prone and would add complexity to the DSL API. If
  required, it is still possible to rely on simple try/catch to have
  specific error handling behavior per node.

- **Sending to a** **DeadLetterQueue**. DeadLetterQueue implementation
  could make sense, but the work would be done in a separate KIP.
  Sending messages to a dead letter queue is possible, but it is up to
  the user  to implement it
