<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
  -->
  
# Observation

Jackrabbit Oak as part of JCR implementation provides support for observing content changes via [EventListener][1].
Event listeners are notified asynchronously, and see events after they occur and the transaction is committed.
`EventListener` can provide a filtering criteria for the type of events they are interested. 

Event processing consume system resources hence its important that `EventListener` can tell Oak precisely which
type of event changes they are interested in. The filtering criteria can be specified in following way

* Via [ObservationManager#addEventListener][2] registration call. 
* Via Jackrabbit extension [JackrabbitEventFilter][3]
* Via Jackrabbit Oak extension [OakEventFilter][4] (new in Oak 1.6)

### Reliable Event Processing

JCR event listeners might miss some events because:

* Event listeners only see events that occurred after they are installed.
  Earlier events are missed (e.g., events via migration).
* In case of power failure (or similar) right after an event was persisted, but before the event listener was able to process it.

To process events reliably, efficiently, and with low latency, consider using a hybrid approach:

* At startup, use a query to process old events (that were missed so far).
* Use an event listener to process recent events with low latency.

For example, to reliably process changes to nodes, one might create an ordered index
on the `jcr:lastModified` property of these nodes.
(This assumes this property is updated when needed.)
At startup of the application, an event listener is registered,
so that subsequent events are processed with low latency.
To process events that happened _before_ the listener was registered
(e.g. via migration, or in case of a crash),
run a query to check for `jcr:lastModified >= $lastProcessed`
where `$lastProcessed` is the time when the last events were processed via event listener.
This timestamp needs to be persisted from time to time
(consider persisting it with a delay of for example one minute, to avoid too frequent updates).

This way, one can be sure all events are processed.

### OakEventFilter

`@since Oak 1.6`

To make use of new filtering capability use following approach


    import org.apache.jackrabbit.api.observation.JackrabbitEvent;
    import org.apache.jackrabbit.api.observation.JackrabbitEventFilter;
    import org.apache.jackrabbit.api.observation.JackrabbitObservationManager;
    import org.apache.jackrabbit.oak.jcr.observation.filter.FilterFactory;
    import org.apache.jackrabbit.oak.jcr.observation.filter.OakEventFilter;
    
    Session session = ...
    EventListener listener = ...
    
    ObservationManager om = session.getWorkspace().getObservationManager();
    
    // Cast to JackrabbitObservationManager
    JackrabbitObservationManager jrom = (JackrabbitObservationManager) om;
    
    // Construct a JackrabbitEventFilter
    JackrabbitEventFilter jrFilter = new JackrabbitEventFilter();
    
    // Wrap it as OakEventFilter
    OakEventFilter oakFilter = FilterFactory.wrap(jrFilter);

    oakFilter.withIncludeSubtreeOnRemove();

    // Set other filtering criteria
    // ...
    
    jrom.addEventListener(listener, oakFilter);

Refer to [OakEventFilter][4] javadocs for more details on what filtering criteria are supported

### Benefits of Filter Approach

In Jackrabbit Oak JCR `EventListener` are implemented as Oak [Observer][5] which are backed by an in memory
queue. Upon any save call done at JCR layer NodeStore in Oak pushes repository root node in this queue which
is later used to compute diff from older root and then generate the JCR Events. 

For events to work properly it needs to be ensure that this in memory queue does not get filled up. Having a precise
filter helps Oak in _prefiltering_ changes before they are added to queue. For e.g. if filter stats that its only
interested in changes under '/content' then Oak can check if changes under that path have happened for given change, 
if not then such a change is not queued.

[1]: https://docs.adobe.com/docs/en/spec/javax.jcr/javadocs/jcr-2.0/javax/jcr/observation/EventListener.html
[2]: https://docs.adobe.com/docs/en/spec/javax.jcr/javadocs/jcr-2.0/javax/jcr/observation/ObservationManager.html
[3]: https://jackrabbit.apache.org/api/2.14/org/apache/jackrabbit/api/observation/JackrabbitEventFilter.html
[4]: https://jackrabbit.apache.org/oak/docs/apidocs/org/apache/jackrabbit/oak/jcr/observation/filter/OakEventFilter.html
[5]: https://jackrabbit.apache.org/oak/docs/apidocs/org/apache/jackrabbit/oak/spi/commit/Observer.html