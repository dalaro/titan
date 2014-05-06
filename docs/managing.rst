Managing Titan
##############

Storage Backends
----------------

Using Cassandra
^^^^^^^^^^^^^^^

.. highlight:: java

.. image:: http://cassandra.apache.org/media/img/cassandra_logo.png
   :target: http://cassandra.apache.org/


.. epigraph::

   The Apache Cassandra database is the right choice when you need scalability and high availability without compromising performance. Linear scalability and proven fault-tolerance on commodity hardware or cloud infrastructure make it the perfect platform for mission-critical data. Cassandra's support for replicating across multiple datacenters is best-in-class, providing lower latency for your users and the peace of mind of knowing that you can survive regional outages. The largest known Cassandra cluster has over 300 TB of data in over 400 machines.

   -- `Apache Cassandra Homepage`_

.. _Apache Cassandra Homepage: http://cassandra.apache.org/

Deploying on Managed Machines
-----------------------------

The following sections outline the various ways in which Titan can be used in concert with Cassandra.

Local Server Mode
^^^^^^^^^^^^^^^^^

.. image:: _static/images/titan-modes-local.png

Cassandra can be run as a standalone database on the same local host as Titan and the end-user application. In this model, Titan and Cassandra communicate with one another via a ``localhost`` socket. Running Titan over Cassandra requires the following setup steps:

#. "`Download Cassandra`, unpack it, and set filesystem paths in ``conf/cassandra.yaml`` and ``conf/log4j-server.properties``
#. Start Cassandra by invoking ``bin/cassandra -f`` on the command line in the directory where Cassandra was unpacked.  Read output to check that Cassandra started successfully.

Now, you can create a Cassandra TitanGraph as follows::

   Configuration conf = new BaseConfiguration();
   conf.setProperty("storage.backend","cassandra");
   conf.setProperty("storage.hostname","127.0.0.1");
   TitanGraph g = TitanFactory.open(conf);

In the Gremlin shell, you can not define the type of the variables ``conf`` and ``g``. Therefore, simply leave off the type declaration.

.. _Download Cassandra:http://cassandra.apache.org/download/

Remote Server Mode
^^^^^^^^^^^^^^^^^^

.. image:: _static/images/titan-modes-distributed.png

When the graph needs to scale beyond the confines of a single machine, then Cassandra and Titan are logically separated into different machines. In this model, the Cassandra cluster maintains the graph representation and any number of Titan instances maintain socket-based read/write access to the Cassandra cluster. The end-user application can directly interact with Titan within the same JVM as Titan.

For example, suppose we have a running Cassandra cluster where one of the machines has the IP address 77.77.77.77, then connecting Titan with the cluster is accomplished as follows (comma separate IP addresses to reference more than one machine)::

Configuration conf = new BaseConfiguration();
conf.setProperty("storage.backend","cassandra");
conf.setProperty("storage.hostname","77.77.77.77");
TitanGraph g = TitanFactory.open(conf);

In the Gremlin shell, you can not define the type of the variables ``conf`` and ``g``. Therefore, simply leave off the type declaration.

Remote Server Mode with Rexster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. image:: _static/images/titan-modes-rexster.png

Rexster can be wrapped around each Titan instance defined in the previous subsection. In this way, the end-user application need not be a Java-based application as it can communicate with Rexster over REST. This type of deployment is great for polyglot architectures where various components written in different languages need to reference and compute on the graph.

.. code-block:: text

   http://rexster.titan.machine1/mygraph/vertices/1
   http://rexster.titan.machine2/mygraph/tp/gremlin?script=g.v(1).out('follows').out('created')

In this case, each Rexster server would be configured to connect to the Cassandra cluster. The following shows the graph specific fragment of the Rexster configuration. Refer to the "Rexster configuration page":Rexster-Graph-Server for a complete example.


.. code-block:: xml

   <graph>
     <graph-name>mygraph</graph-name>
     <graph-type>com.thinkaurelius.titan.tinkerpop.rexster.TitanGraphConfiguration</graph-type>
     <graph-location></graph-location>
     <graph-read-only>false</graph-read-only>
     <properties>
       <storage.backend>cassandra</storage.backend>
       <storage.hostname>77.77.77.77</storage.hostname>
     </properties>
     <extensions>
       <allows>
         <allow>tp:gremlin</allow>
       </allows>
     </extensions>
   </graph>

Titan Embedded Mode
^^^^^^^^^^^^^^^^^^^

.. image:: _static/images/titan-modes-embedded.png

Finally, Cassandra can be embedded in Titan, which means, that Titan and Cassandra run in the same JVM and communicate via in process calls rather than over the network. This removes the (de)serialization and network protocol overhead and can therefore lead to considerable performance improvements. In this deployment mode, Titan internally starts a cassandra daemon and Titan no longer connects to an existing cluster but is its own cluster.

To use Titan in embedded mode, simply configure ``embeddedcassandra`` as the storage backend. The configuration options listed below also apply to embedded Cassandra. In creating a Titan cluster, ensure that the individual nodes can discover each other via the Gossip protocol, so setup a Titan-with-Cassandra-embedded cluster much like you would a stand alone Cassandra cluster. When running Titan in embedded mode, the Cassandra yaml file is configured using the additional configuration option ``storage.cassandra-config-dir``, which specifies the yaml file as a full url, e.g. ``storage.cassandra-config-dir = file:///home/cassandra.yaml``.

When running a cluster with Titan and Cassandra embedded, it is advisable to expose Titan through the Rexster server so that applications can remotely connect to the Titan graph database and execute queries.

Note, that running Titan with Cassandra embedded requires GC tuning. While embedded Cassandra can provide lower latency query answering, its GC behavior under load is less predictable.

Cassandra Specific Configuration
--------------------------------

In addition to the general "Titan Graph Configuration":Graph-Configuration, there are the following Cassandra specific Titan configuration options:

.. Emacs automatically recognizes this as a table and handles the tedious
.. reformatting automatically.  vim can also automatically reformat this
.. table by way of its "tabular" addon.

+--------------------------------------+------------------------------------------+--------+-------+----------+
|Option                                |Description                               |Value   |Default|Modifiable|
+======================================+==========================================+========+=======+==========+
|storage.hostname                      |IP address or hostname of the Cassandra   |IP      |-      |Yes       |
|                                      |cluster node that this Titan instance     |address |       |          |
|                                      |connects to. Use a list of comma-separated|or      |       |          |
|                                      |hostnames or IP addresses to seed multiple|hostname|       |          |
|                                      |multiple cassandra nodes                  |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.port                          |Port on which to connect to Cassandra     |Integer |9160   |Yes       |
|                                      |cluster node                              |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.connection-timeout            |Default time out in milliseconds after    |Integer |10000  |Yes       |
|                                      |which to fail a connection attempt with a |        |       |          |
|                                      |Cassandra node                            |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.connection-pool-size          |Maximum size of the connection pool for   |Integer |32     |Yes       |
|                                      |connections to the Cassandra cluster      |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.read-consistency-level        |Cassandra consistency level for read      |String  |QUORUM |Yes       |
|                                      |operations                                |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.write-conssistency-level      |Cassandra consistency level for write     |String  |QUORUM |Yes       |
|                                      |operations                                |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.replication-factor            |The replication factor to use. The higher |Integer |1      |No        |
|                                      |the replication factor, the more robust   |        |       |          |
|                                      |the graph database is to machine failure  |        |       |          |
|                                      |at the expense of data duplication. *The  |        |       |          |
|                                      |default value should be overwritten for   |        |       |          |
|                                      |production system to ensure robustness. A |        |       |          |
|                                      |value of 3 is recommended.* This          |        |       |          |
|                                      |replication factor can only be set when   |        |       |          |
|                                      |the keyspace is initially created. **On an|        |       |          |
|                                      |existing keyspace, this value is          |        |       |          |
|                                      |ignored.**                                |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+
|storage.cassandra.thrift.frame_size_mb|The maximum frame size to be used by      |Integer |16     |No        |
|                                      |thrift for transport. Increase this value |        |       |          |
|                                      |when retrieving very large result         |        |       |          |
|                                      |sets. **Only applicable when              |        |       |          |
|                                      |storage.backend=cassandrathrift**         |        |       |          |
+--------------------------------------+------------------------------------------+--------+-------+----------+

For more information on Cassandra consistency levels and acceptable values, please refer to the `Cassandra Thrift API`_. In general, higher levels are more consistent and robust but have higher latency.

.. _Cassandra Thrift API: http://wiki.apache.org/cassandra/API10

Global Graph Operations
-----------------------

Titan over Cassandra supports global vertex and edge iteration. However, note that all these vertices and/or edges will be loaded into memory which can cause ``OutOfMemoryException``. Use `Faunus`_ to iterate over all vertices or edges in large graphs.

.. _Faunus: http://faunus.thinkaurelius.com/

Deploying on Amazon EC2
-----------------------

.. image:: http://cdn001.practicalclouds.com/user-content/1_Dave%20McCormick//logos/Amazon%20AWS%20plus%20EC2%20logo_scaled.png
   :target: http://aws.amazon.com/ec2/

bq. `Amazon EC2`_ is a web service that provides resizable compute capacity in the cloud. It is designed to make web-scale computing easier for developers.

Follow these steps to setup a Cassandra cluster on EC2 and deploy Titan over Cassandra. To follow these instructions, you need an Amazon AWS account with established authentication credentials and some basic knowledge of AWS and EC2.

.. _Amazon EC2: http://aws.amazon.com/ec2/

Setup Cassandra Cluster
^^^^^^^^^^^^^^^^^^^^^^^

These instructions for configuring and launching the DataStax Cassandra Community Edition AMI are based on the `DataStax AMI Docs`_ and focus on aspects relevant for a Titan deployment.

.. _DataStax AMI Docs: http://www.datastax.com/docs/datastax_enterprise2.0/ami/install_dse_ami

Setting up Security Group
^^^^^^^^^^^^^^^^^^^^^^^^^

* Navigate to the EC2 Console Dashboard, then click on "Security Groups" under "Network & Security".

* Create a new security group. Click Inbound.  Set the "Create a new rule" dropdown menu to "Custom TCP rule".  Add a rule for port 22 from source 0.0.0.0/0.  Add a rule for ports 1024-65535 from the security group members.  If you don't want to open all unprivileged ports among security group members, then at least open 7000, 7199, and 9160 among security group members.  Tip: the "Source" dropdown will autocomplete security group identifiers once "sg" is typed in the box, so you needn't have the exact value ready beforehand.

Launch DataStax Cassandra AMI
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* "Launch the `DataStax AMI`_ in your desired zone 

.. _DataStax AMI: https://aws.amazon.com/amis/datastax-auto-clustering-ami-2-2

* On the Instance Details page of the Request Instances Wizard, set "Number of Instances" to your desired number of Cassandra nodes. Set "Instance Type" to at least m1.large. We recommend m1.large.

* On the Advanced Instance Options page of the Request Instances Wizard, set the "as text" radio button under "User Data", then fill this into the text box:

.. code-block:: text

   --clustername [cassandra-cluster-name]
   --totalnodes [number-of-instances]
   --version community 
   --opscenter no

[number-of-instances] in this configuration must match the number of EC2 instances configured on the previous wizard page. [cassandra-cluster-name] can be any string used for identification. For example:

.. code-block:: text

   --clustername titan
   --totalnodes 4
   --version community 
   --opscenter no

* On the Tags page of the Request Instances Wizard you can apply any desired configurations. These tags exist only at the EC2 administrative level and have no effect on the Cassandra daemons' configuration or operation.

* On the Create Key Pair page of the Request Instances Wizard, either select an existing key pair or create a new one.  The PEM file containing the private half of the selected key pair will be required to connect to these instances.

* On the Configure Firewall page of the Request Instances Wizard, select the security group created earlier.

* Review and launch instances on the final wizard page.

Verify Successful Instance Launch
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* SSH into any Cassandra instance node: ``ssh -i [your-private-key].pem ubuntu@[public-dns-name-of-any-cassandra-instance]``

* Run the Cassandra nodetool ``nodetool -h 127.0.0.1 ring`` to inspect the state of the Cassandra token ring.  You should see as many nodes in this command's output as instances launched in the previous steps.

Note, that the AMI takes a few minutes to configure each instance. A shell prompt will appear upon successful configuration when you SSH into the instance.

Launch Titan Instances
^^^^^^^^^^^^^^^^^^^^^^

Launch additional EC2 instances to run Titan which are either configured in Remote Server Mode or Remote Server Mode with Rexster as described above. You only need to note the IP address of one of the Cassandra cluster instances and configure it as the host name. The particular EC2 instance to run and the particular configuration depends on your use case. 

Example Titan Instance on Amazon Linux AMI
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Launch the `Amazon Linux AMI`_ in the same zone of the Cassandra cluster. Choose your desired EC2 instance type depending on the amount of resources you need. Use the default configuration options and select the same Key Pair and Security Group as for the Cassandra cluster configured in the previous step.

* SSH into the newly created instance via ``ssh -i [your-private-key].pem ec2-user@[public-dns-name-of-the-instance]``. You may have to wait a little for the instance to launch.

* `Download`_ the current Titan distribution with ``wget`` and unpack the archive locally to the home directory. Start the gremlin shell to verify that Titan runs successfully. For more information on how to unpack Titan and start the gremlin shell, please refer to the "Getting Started guide":Getting-Started.

* Create a configuration file with ``vi titan.properties`` and add the following lines::

      storage.backend = cassandra
      storage.hostname = [IP-address-of-one-Cassandra-EC2-instance]

You may add additional configuration options found on this page or under "Graph Configuration":Graph-Configuration.

* Start the gremlin shell again and type the following::

      gremlin> g = TitanFactory.open('titan.properties')              
      ==>titangraph[cassandra:[IP-address-of-one-Cassandra-EC2-instance]]

You have successfully connected this Titan instance to the Cassandra cluster and can start to operate on the graph.


.. _Amazon Linux AMI: http://aws.amazon.com/amazon-linux-ami
.. _Download: https://github.com/thinkaurelius/titan/wiki/Downloads

Connect to Cassandra cluster in EC2 from outside EC2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Opening the usual Cassandra ports (9160, 7000, 7199) in the security group is not enough, because the Cassandra nodes by default broadcast their ec2-internal IPs, and not their public-facing IPs.

The resulting behavior is that you can open a Titan graph on the cluster by connecting to port 9160 on any Cassandra node, but all requests to that graph time out.  This is because Cassandra is telling the client to connect to an unreachable IP.

To fix this, set the "broadcast-address" property for each instance in /etc/cassandra/cassandra.yaml to its public-facing IP, and restart the instance.  Do this for all nodes in the cluster.  Once the cluster comes back, nodetool reports the correct public-facing IPs to which connections from the local machine are allowed.

Changing the "broadcast-address" property allows you to connect to the cluster from outside ec2, but it might also mean that traffic originating within ec2 will have to round-trip to the internet and back before it gets to the cluster.  So, this approach is only useful for development and testing.


TODO Sections from Github Wiki
==============================

*** [[Using HBase]]
*** [[Using Persistit]]
*** [[Using BerkeleyDB]]
** Indexing Backends
*** [[Indexing Backend Overview]]
*** [[Using Elastic Search]]
*** [[Using Lucene]]
*** [[Full Text and String Search]]
*** [[Direct Index Query]]
** Type Management
*** [[Type Definition Overview]] (*cheat sheet*)
*** [[Vertex-Centric Indices]]
** Configuration and Tuning
*** [[Graph Configuration]] (*cheat sheet*)
*** [[Datatype and Attribute Serializer Configuration]]
*** [[Example Graph Configuration]]
** [[Data Caching]]
*** [[Database Cache]]
*** [[Transaction Cache]]
