Using Titan
###########

.. highlight:: java

TinkerPop
=========

Titan natively implements the Blueprints Interface which means that it supports all of the open-source technologies in the `TinkerPop`_ graph stack:

* `Blueprints`_ - The property graph model interface implemented by Titan which provides various utilities to aid developers.
* `Gremlin`_ - A graph traversal language for expressing complex walks through a graph.  `GremlinDocs`_ is a cheat-sheet for the language.
* `Frames`_ - An object-to-graph mapper for rendering Java objects from graph data.
* `Rexster`_ - A graph server for exposing the graph via REST, a binary protocol, and HTML-based GUI tools.

Being a native implementation means that Titan directly implements the Blueprints interface without an adapter. This makes Titan one of the most efficient Blueprints implementations which benefits the performance of all TinkerPop projects when running on Titan.

.. _TinkerPop: http://www.tinkerpop.com
.. _Blueprints: http://blueprints.tinkerpop.com
.. _Gremlin: http://gremlin.tinkerpop.com
.. _GremlinDocs: http://gremlindocs.com/
.. _Frames: http://frames.tinkerpop.com
.. _Rexster: http://rexster.tinkerpop.com


Transactions
============

Transaction Handling
--------------------

Every graph operation in Titan occurs within the context of a transaction. According to the Blueprints' specification, each thread opens its own transaction against the graph database with the first operation (i.e. retrieval or mutation) on the graph::

   TitanGraph g = TitanFactory.open("/tmp/titan");
   Vertex juno = g.addVertex(null); //Automatically opens a new transaction
   juno.setProperty("name", "juno");
   g.commit(); //Commits transaction

In this example, a local Titan graph database is opened. Adding the vertex "juno" is the first operation (in this thread) which automatically opens a new transaction. All subsequent operations occur in the context of that same transaction until the transaction is explicitly stopped or the graph database ``shutdown()`` which commits all currently running transactions. Note, that both read and write operations occur within the context of a transaction.

Transactional Scope
^^^^^^^^^^^^^^^^^^^

All graph elements (vertices, edges, and types) are associated with the transactional scope in which they were retrieved or created. Under Blueprint's default transactional semantics, transactions are automatically created with the first operation on the graph and closed explicitly using ``commit()`` or ``rollback()``. Once the transaction is closed, all graph elements associated with that transaction become stale and unavailable. However, Titan will automatically transition vertices and types into the new transactional scope as shown in this example::

   TitanGraph g = TitanFactory.open("/tmp/titan");
   Vertex juno = g.addVertex(null); //Automatically opens a new transaction
   g.commit(); //Ends transaction
   juno.setProperty("name", "juno"); //Vertex is automatically transitioned

Edges, on the other hand, are not automatically transitioned and cannot be accessed outside their original transaction. They must be explicitly transitioned.

::

   Edge e = juno.addEdge("knows",g.addVertex(null));
   g.commit(); //Ends transaction
   e = g.getEdge(e); //Need to refresh edge
   e.setProperty("time", 99);

Transaction Failures
^^^^^^^^^^^^^^^^^^^^

When committing a transaction, Titan will attempt to persist all changes to the storage backend. This might not always be successful due to IO exceptions, network errors, machine crashes or resource unavailability. Hence, transactions can fail. In fact, transactions *will eventually fail* in sufficiently large systems. Therefore, we highly recommend that your code expects and accommodates such failures.

::

   try {
       if (g.getVertices("name",name).iterator().hasNext())
           throw new IllegalArgumentException("Username already taken: " + name);
       Vertex user = g.addVertex(null);
       user.setProperty("name", name);
       g.commit();
   } catch (TitanException e) {
       //Recover, retry, or return error message
   }

The example above demonstrates a simplified user signup implementation where ``name`` is the name of the user who wishes to register. First, it is checked whether a user with that name already exists. If not, a new user vertex is created and the name assigned. Finally, the transaction is committed.

If the transaction fails, a ``TitanException`` is thrown. There are a variety of reasons why a transaction may fail. Titan differentiates between _potentially temporary_ and _permanent_ failures. 

Potentially temporary failures are those related to resource unavailability and IO hickups (e.g. network timeouts). Titan automatically tries to recover from temporary failures by retrying to persist the transactional state after some delay. The number of retry attempts and the retry delay can be configured through the [[Titan graph configuration|Graph Configuration]].

Permanent failures can be caused by complete connection loss, hardware failure or lock contention. To understand the cause of lock contention, consider the signup example above and suppose a user tries to signup with username "juno". That username may still be available at the beginning of the transaction but by the time the transaction is committed, another user might have concurrently registered with "juno" as well and that transaction holds the lock on the username therefore causing the other transaction to fail. Depending on the transaction semantics one can recover from a lock contention failure by re-running the entire transaction.

Permanent exceptions that can fail a transaction include:

* PermanentLockingException(*Local lock contention*): Another local thread has already been granted a conflicting lock.
* PermanentLockingException(*Expected value mismatch for X: expected=Y vs actual=Z*): The verification that the value read in this transaction is the same as the one in the datastore after applying for the lock failed. In other words, another transaction modified the value after it had been read and modified.

Gotchas
^^^^^^^

* Transactions are started automatically with the first operation executed against the graph. One does NOT have to start a transaction manually. The method ``newTransaction`` is used to start [[multi threaded transactions]] only.

* Transactions are automatically started under the Blueprints semantics but *not* automatically terminated. Transactions have to be terminated manually with ``g.commit()`` if successful or ``g.rollback()`` if not. Manual termination of transactions is necessary because only the user knows the transactional boundary. 
A transaction will attempt to maintain its state from the beginning of the transaction. This might lead to unexpected behavior in multi-threaded applications as illustrated in the following artificial example::

   v = g.v(4) //Retrieve vertex, first action automatically starts transaction
   v.bothE
   >> returns nothing, v has no edges
   //thread is idle for a few seconds, another thread adds edges to v
   v.bothE
   >> still returns nothing because the transactional state from the beginning is maintained

Such unexpected behavior is likely to occur in client-server applications where the server maintains multiple threads to answer client requests. It is therefore important to terminate the transaction after a unit of work (e.g. code snippet, query, etc). For instance, `Rexster`_ manages the transactional boundary for each gremlin query. So, the example above should be::

   v = g.v(4) //Retrieve vertex, first action automatically starts transaction
   v.bothE
   g.commit()
   //thread is idle for a few seconds, another thread adds edges to v
   v.bothE
   >> returns the newly added edge
   g.commit()

Next Steps
^^^^^^^^^^

* Read more about `Blueprints Transactions`_

.. _Blueprints Transactions: https://github.com/tinkerpop/blueprints/wiki/Graph-Transactions


.. _multi-threaded-tx:

Multi-Threaded Transactions
---------------------------

Titan supports multi-threaded transactions through Blueprint's ``ThreadedTransactionalGraph`` interface. Hence, to speed up transaction processing and utilize multi-core architectures multiple threads can run concurrently in a single transaction.

With Blueprints' default [[transaction handling]] each thread automatically opens its own transaction against the graph database. To open a thread-independent transaction, use the ``newTransaction()`` method.

::

   TransactionalGraph tx = g.newTransaction();
   Thread[] threads = new Thread[10];
   for (int i=0;i<threads.length;i++) {
       threads[i]=new Thread(new DoSomething(tx));
       threads[i].start();
   }
   for (int i=0;i<threads.length;i++) threads[i].join();
   tx.commit();

The ``newTransaction()`` method returns a new ``TransactionalGraph`` object that represents this newly opened transaction. The graph object ``tx`` supports all of the method that the original graph did, but does so without opening new transactions for each thread. This allows us to start multiple threads which all do-something in the same transaction and finally commit the transaction when all threads have completed their work.

Titan relies on optimized concurrent data structures to support hundreds of concurrent threads running efficiently in a single transaction.

Concurrent Algorithms
^^^^^^^^^^^^^^^^^^^^^

Thread independent transactions started through ``newTransaction()`` are particularly useful when implementing concurrent graph algorithms. Most traversal or message-passing (ego-centric) like graph algorithms are `embarrassingly parallel`_ which means they can be parallelized and executed through multiple threads with little effort. Each of these threads can operate on a single ``TransactionalGraph`` object returned by ``newTransaction`` without blocking each other.

.. _embarrassingly parallel: http://en.wikipedia.org/wiki/Embarrassingly_parallel 

Nested Transactions
^^^^^^^^^^^^^^^^^^^

Another use case for thread independent transactions is nested transactions that ought to be independent from the surrounding transaction.

For instance, assume a long running transactional job that has to create a new vertex with a unique name. Since enforcing unique names requires the acquisition of a lock (see [[Type Definition Overview]] for more detail) and since the transaction is running for a long time, lock congestion and expensive transactional failures are likely.

::

   Vertex v1 = g.addVertex(null);
   //Do many other things
   Vertex v2 = g.addVertex(null);
   v2.setProperty("uniqueName","foo");
   g.addEdge(null,v1,v2,"related");
   //Do many other things
   g.commit(); // Likely to fail due to lock congestion

One way around this is to create the vertex in a short, nested thread-independent transaction as demonstrated by the following pseudo code::

   Vertex v1 = g.addVertex(null);
   //Do many other things
   TransactionalGraph tx = g.newTransaction();
   Vertex v2 = tx.addVertex(null);
   v2.setProperty("uniqueName","foo");
   tx.commit();
   g.addEdge(null,v1,g.getVertex(v2),"related"); //Need to load v2 into outer transaction
   //Do many other things
   g.commit(); // Likely to fail due to lock congestion

Gotchas
^^^^^^^

When using multi-threaded transactions via ``newTransaction`` all vertices and edges retrieved or created in the scope of that transaction are *not* available outside the scope of that transaction. Accessing such elements after the transaction has been closed will result in an exception. As demonstrated in the example above, such elements have to be explicitly refreshed in the new transaction using ``g.getVertex(existingVertex)`` or ``g.getEdge(existingEdge)``.

Next steps
^^^^^^^^^^

Read more about `Blueprints's ThreadedTransactionalGraph`_.

.. _Blueprints's ThreadedTransactionalGraph: https://github.com/tinkerpop/blueprints/wiki/Graph-Transactions.

Transaction Configuration
-------------------------

Titan's ``TitanGraph.buildTransaction()`` method gives the user the ability to configure and start a new :ref:`multi-threaded transaction <multi-threaded-tx>` against a ``TitanGraph``. Hence, it is identical to ``TitanGraph.newTransaction()`` with additional configuration options.

``buildTransaction()`` returns a ``TransactionBuilder`` which allows the following aspects of a transaction to be configured:

* ``readOnly()`` - makes the transaction read-only and any attempt to modify the graph will result in an exception.
* ``enableBatchLoading()`` - enables batch-loading for an individual transaction. This setting results in similar efficiencies as the graph-wide setting ``storage.batch-loading`` due to the disabling of consistency checks and other optimizations. Unlike ``storage.batch-loading`` this option will not change the behavior of the storage backend.
* ``setTimestamp(long)`` - Sets the timestamp for this transaction as communicated to the storage backend for persistence. Depending on the storage backend, this setting may be ignored. For eventually consistent backends, this is the timestamp used to resolve write conflicts. If this setting is not explicitly specified, Titan uses the current time.
* ``setCacheSize(long size)`` - The number of vertices this transaction caches in memory. The larger this number, the more memory a transaction can potentially consume. If this number is too small, a transaction might have to re-fetch data which causes delays in particular for long running transactions.
* ``checkInternalVertexExistence()`` - Whether this transaction should double-check the existence of vertices during query execution. This can be useful to avoid *phantom vertices* on eventually consistent storage backends. Disabled by default. Enabling this setting can slow down query processing.

Once, the desired configuration options have been specified, the new transaction is started via ``start()`` which returns a ``TitanTransaction``.


Gotchas and Limitations
=======================

[[images/titan-head.png|float|align=left]] There are various limitations and "gotchas" that one should be aware of when using Titan. Some of these limitations are necessary design choices and others are issues that will be rectified as Titan development continues. Finally, the last section provides solutions to common issues.

Design Limitations
------------------

Size Limitation
^^^^^^^^^^^^^^^

Titan can store up to a quintillion edges (2^60) and half as many vertices. That limitation is imposed by Titan's id scheme.

DataType Definitions
^^^^^^^^^^^^^^^^^^^^

When declaring the data type of a property key using ``dataType(Class)`` Titan will enforce that all properties for that key have the declared type, unless that type is ``Object.class``. This is an equality type check, meaning that sub-classes will not be allowed. For instance, one cannot declare the data type to be ``Number.class`` and use ``Integer`` or ``Long``. For efficiency reasons, the type needs to match exactly. Hence, use ``Object.class`` as the data type for type flexibility. In all other cases, declare the actual data type to benefit from increased performance and type safety.

Edge Retrievals are not O(1)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Retrieving an edge by id, e.g ``tx.getEdge(edge.getId())``, is not a constant time operation. Titan will retrieve an adjacent vertex of the edge to be retrieved and then execute a vertex query to identify the edge. The former is constant time but the latter is potentially linear in the number of edges incident on the vertex with the same edge label.

This also applies to index retrievals for edges via a standard or external index.

Temporary Limitations
---------------------

Key Index Must Be Created Prior to Key Being Used
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To index vertices or edges by key, the respective key index must be created before the key is first used in a vertex or edge property. Read more about creating [[vertex indexes|Blueprints Interface]].

Unable to Drop Key Indices
^^^^^^^^^^^^^^^^^^^^^^^^^^

Once an index has been created for a key, it can never be removed. 

Types Can Not Be Changed Once Created
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This pitfall constrains the graph schema. While the graph schema can be extended, previous declarations cannot be changed. 

Batch Loading Speed
^^^^^^^^^^^^^^^^^^^

Titan provides a batch loading mode that can be enabled through the `configuration <Graph Configuration_>`_ (TODO link to graph configuration section when populated). However, this batch mode only facilitates faster loading into the storage backend, it does not use storage backend specific batch loading techniques that prepare the data in memory for disk storage. As such, batch loading in Titan is currently slower than batch loading modes provided by single machine databases. The [[Bulk Loading]] documentation lists ways to speed up batch loading in Titan.

Another limitation related to batch loading is the failure to load millions of edges into a single vertex at once or in a short time of period. Such *supernode loading* can fail for some storage backends. This limitation also applies to dense index entries. For more information, please refer to `Issue #11 <issue 11_>`_.

.. _issue 11: https://github.com/thinkaurelius/titan/issues/11

Beware
------

Multiple Titan instances on one machine
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Running multiple Titan instances on one machine backed by the *same* storage backend (distributed or local) requires that each of these instances has a unique configuration for ``storage.machine-id-appendix``. Otherwise, these instances might overwrite each other leading to data corruption. See [[Graph Configuration]] for more information.

Accidental type creation
^^^^^^^^^^^^^^^^^^^^^^^^

By default, Titan will automatically create property keys and edge labels when a new type is encountered. It is strongly encouraged that users explicitly [[define types|Type-Definition-Overview]] and disable automatic type creation by setting the [[graph configuration|Graph-Configuration]] option ``autotype = none``.

Custom Class Datatype
^^^^^^^^^^^^^^^^^^^^^

Titan supports arbitrary objects as attribute values on properties. To use a custom class as data type in Titan, either register a custom serializer or ensure that the class has a no-argument constructor and implements the ``equals`` method because Titan will verify that it can successfully de-/serialize objects of that class. Please read [[Datatype and Attribute Serializer Configuration]] for more information.

Transactional Scope for Edges
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Edges should not be accessed outside the scope in which they were originally created or retrieved.

Locking Exceptions
^^^^^^^^^^^^^^^^^^

When defining unique [[Titan types|Type Definition Overview]] with locking enabled (i.e. requesting that Titan ensures uniqueness) it is likely to encounter locking exceptions of the type ``PermanentLockingException`` under concurrent modifications to the graph.

Such exceptions are to be expected, since Titan cannot know how to recover from a transactional state where an earlier read value has been modified by another transaction since this may invalidate the state of the transaction. It most cases it is sufficient to simply re-run the transaction. If locking exceptions are very frequent, try to analyze and remove the source of congestion.

Double and Float Data Types
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Titan internally represents ``Double`` and ``Float`` data types as fixed decimal numbers. Doubles are stored with up to 6 decimal digits and floats with up to 3. This representation enables range retrievals in vertex centric queries. However, it significantly limits the precision and range of doubles and floats. 
Use ``FullDouble`` and ``FullFloat`` as data type to get the full precision of floating point numbers. However, note that these data types cannot be used in range-constrained vertex centric queries.

Ghost Vertices
^^^^^^^^^^^^^^

When the same vertex is concurrently removed in one transaction and modified in another, both transactions will successfully commit on eventually consistent storage backends and the vertex will still exist with only the modified properties or edges. This is referred to as a ghost vertex. It is possible to guard against ghost vertices on eventually consistent backends using key [[out-uniqueness|Type Definition Overview]] but this is prohibitively expensive in most cases. A more scalable approach is to allow ghost vertices temporarily and clearing them out in regular time intervals, for instance using `Titan tools`_.

.. Titan tools: https://github.com/StartTheShift/titan-tools

Another option is to detect them at read-time using the [[transaction configuration|Transaction-Configuration]] option ``checkInternalVertexExistence()``

Snappy 1.4 does not work with Java 1.7
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Cassandra 1.2.x makes use of Snappy 1.4. Titan will not be able to connect to Cassandra if the server is running Java 1.7 and Cassandra 1.2.x (with Snappy 1.4). Be sure to remove the Snappy 1.4 jar in the ``cassandra/lib`` directory and replace with a `Snappy 1.5 jar version`_.

.. _Snappy 1.5 jar version: http://code.google.com/p/snappy-java/downloads/list

Debug-level Logging
^^^^^^^^^^^^^^^^^^^

When the log level is set to ``debug`` Titan produces *a lot* of logging output which is useful to understand how particular queries get compiled, optimized, and executed. However, the output is so large that it will impact the query performance noticeably. Hence, you ``info`` or above for production systems or benchmarking.

Useful Tips
^^^^^^^^^^^

Titan OutOfMemoryException or excessive Garbage Collection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you experience memory issues or excessive garbage collection while running Titan it is likely that the caches are configured incorrectly. If the caches are too large, the heap may fill up with cache entries. Try reducing the size of the transaction level cache before tuning the database level cache, in particular if you have many concurrent transactions. Read more about [[Titan's caching layers | Data Caching]].

Removing JAMM Warning Messages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When launching Titan with embedded Cassandra, the following warnings may be displayed:

``958 [MutationStage:25] WARN  org.apache.cassandra.db.Memtable  - MemoryMeter uninitialized (jamm not specified as java agent); assuming liveRatio of 10.0.  Usually this means cassandra-env.sh disabled jamm because you are using a buggy JRE; upgrade to the Sun JRE instead``

Cassandra uses a Java agent called ``MemoryMeter`` which allows it to measure the actual memory use of an object, including JVM overhead.  To use `JAMM`_ (Java Agent for Memory Measurements), the path to the JAMM jar must be specific in the Java javaagent parameter when launching the JVM (e.g. ``-javaagent:path/to/jamm.jar``). Rather than modifying ``titan.sh`` and adding the javaagent parameter, I prefer to set the ``JAVA_OPTIONS`` environment variable with the proper javaagent setting:

.. code-block:: bash

   export JAVA_OPTIONS=-javaagent:$TITAN_HOME/lib/jamm-0.2.5.jar

.. _JAMM: https://github.com/jbellis/jamm

Cassandra Connection Problem
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

By default, Titan uses the Astyanax library to connect to Cassandra clusters. On EC2 and Rackspace, it has been reported that Astyanax was unable to establish a connection to the cluster. In those cases, changing the backend to ``storage.backend=cassandrathrift`` solved the problem.

ElasticSearch OutOfMemoryException
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When numerous clients are connecting to ElasticSearch, it is likely that an ``OutOfMemoryException`` occurs. This is not due to a memory issue, but to the OS not allowing more threads to be spawned by the user (the user running ElasticSearch). To circumvent this issue, increase the number of allowed processes to the user running ElasticSearch. For example, increase the ``ulimit -u`` from the default 1024 to 10024.
