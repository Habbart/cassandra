.. Licensed to the Apache Software Foundation (ASF) under one
.. or more contributor license agreements.  See the NOTICE file
.. distributed with this work for additional information
.. regarding copyright ownership.  The ASF licenses this file
.. to you under the Apache License, Version 2.0 (the
.. "License"); you may not use this file except in compliance
.. with the License.  You may obtain a copy of the License at
..
..     http://www.apache.org/licenses/LICENSE-2.0
..
.. Unless required by applicable law or agreed to in writing, software
.. distributed under the License is distributed on an "AS IS" BASIS,
.. WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
.. See the License for the specific language governing permissions and
.. limitations under the License.

Denylisting Partitions
----------------------

Due to access patterns and data modeling, sometimes there are specific partitions that are "hot" and can cause instability in a Cassandra cluster. This often occurs when your data model includes many update or insert operations on a single partition, causing the partition to grow very large over time and in turn making it very expensive to read and maintain.

Cassandra supports "denylisting" these problematic partitions so that when clients issue point reads (`SELECT` statements with the partition key specified) or range reads (`SELECT *`, etc that pull a range of data) that intersect with a blocked partition key, the query will be immediately rejected with an `InvalidQueryException`.

How to denylist a partition key
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The ``system_distributed.denylisted_partitions`` table can be used to denylist partitions. There are a couple of ways to interact with and mutate this data. First: directly via CQL by inserting a record with the following details:

- Keyspace name (ks_name)
- Table name (table_name)
- Partition Key (partition_key)

The partition key format needs to be in the same form required by ``nodetool getendpoints``.

Following are several examples for denylisting partition keys in keyspace `ks` and table `table1` for different data types on the primary key `Id`:

 - Id is a simple type - INSERT INTO system_distributed.denylisted_partitions (ks_name, table_name, partition_key) VALUES ('ks','table1','1');
 - Id is a blob        - INSERT INTO system_distributed.denylisted_partitions (ks_name, table_name, partition_key) VALUES ('ks','table1','12345f');
 - Id has a colon      - INSERT INTO system_distributed.denylisted_partitions (ks_name, table_name, partition_key) VALUES ('ks','table1','1\:2');

In the case of composite column partition keys (Key1, Key2):

 - INSERT INTO system_distributed.denylisted_partitions (ks_name, table_name, partition_key) VALUES ('ks', 'table1', 'k11:k21')

Special considerations
^^^^^^^^^^^^^^^^^^^^^^
The denylist has the property in that you want to keep your cache (see below) and CQL data on a replica set as close together as possible so you don't have different nodes in your cluster denying or allowing different keys. To best achieve this, the workflow for a denylist change (addition or deletion) should `always be as follows`:

JMX PATH (preferred for single changes):

1. Call the JMX hook for ``denylistKey()`` with the desired key
2. Double check the cache reloaded with ``isKeyDenylisted()``
3. Check for warnings about unrecognized keyspace/table combinations, limits, or consistency level. If you get a message about nodes being down and not hitting CL for denylist, recover the downed nodes and then trigger a reload of the cache on each node with ``loadPartitionDenylist()``

CQL PATH (preferred for bulk changes):

1. Mutate the denylisted partition lists via CQL
2. Trigger a reload of the denylist cache on each node via JMX ``loadPartitionDenylist()`` (see below)
3. Check for warnings about lack of availability for a denylist refresh. In the event nodes are down, recover them, then go to 2.

Due to conditions on known unavailable range slices leading to alert storming on startup, the denylist cache won't load on node start unless it can achieve the configured consistency level in cassandra.yaml, `denylist_consistency_level`. The JMX call to `loadPartitionDenylist` will, however, load the cache regardless of the number of nodes available. This leaves the control for denylisting or not denylisting during degraded cluster states in the hands of the operator.

Denylisted Partitions Cache
^^^^^^^^^^^^^^^^^^^^^^^^^^^
Cassandra internally maintains an on-heap cache of denylisted partitions loaded from ``system_distributed.denylisted_partitions``. The values for a table will be automatically repopulated every ``denylist_refresh_seconds`` as specified in the `conf/cassandra.yaml` file, defaulting to 600 seconds, or 10 minutes. Invalid records (unknown keyspaces, tables, or keys) will be ignored and not cached on load.

The cache can be refreshed in the following ways:

- During Cassandra node startup
- Via the automatic on-heap cache refresh mechanisms. Note: this will occur asynchronously on query after the ``denylist_refresh_seconds`` time is hit.
- Via the JMX command: ``loadPartitionDenylist`` in ``the org.apache.cassandra.service.StorageProxyMBean`` invocation point.

The Cache size is bounded by the following two config properties

- denylist_max_keys_per_table
- denylist_max_keys_total

On cache load, if a table exceeds the value allowed in `denylist_max_keys_per_table`, a warning will be printed to the logs and the remainder of the keys will not be cached. Similarly, if the total allowable size is exceeded subsequent ks_name + table_name combinations (in clustering / lexicographical order) will be skipped as well, and a warning logged to the server logs.

Note: given the required workflow of 1) Mutate, 2) Reload cache, the auto-reload property seems superfluous. It exists to ensure that, should an operator make a mistake and denylist (or undenylist) a key but forget to reload the cache, that intent will be captured on the next cache reload.

JMX Interface
^^^^^^^^^^^^^

+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| Command                                                                    | Effect                                                                          |
+============================================================================+=================================================================================+
| loadPartitionDenylist()                                                    | Reloads cached denylist from CQL table                                          |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| getPartitionDenylistLoadAttempts()                                         | Gets the count of cache reload attempts                                         |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| getPartitionDenylistLoadSuccesses()                                        | Gets the count of cache reload successes                                        |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| setEnablePartitionDenylist(boolean enabled)                                | Enables or disables the partition denylisting functionality                     |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| setEnableDenylistWrites(boolean enabled)                                   | Enables or disables write denylisting functionality                             |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| setEnableDenylistReads(boolean enabled)                                    | Enables or disables read denylisting functionality                              |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| setEnableDenylistRangeReads(boolean enabled)                               | Enables or disables range read denylisting functionality                        |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| denylistKey(String keyspace, String table, String partitionKeyAsString)    | Adds a specific keyspace, table, and partition key combo to the denylist        |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| removeDenylistKey(String keyspace, String cf, String partitionKeyAsString) | Removes a specific keyspace, table, and partition key combo from the denylist   |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| setDenylistMaxKeysPerTable(int value)                                      | Limits count of allowed keys per table in the denylist                          |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| setDenylistMaxKeysTotal(int value)                                         | Limits the total count of allowable denylisted keys in the system               |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
| isKeyDenylisted(String keyspace, String table, String partitionKeyAsString)| Indicates whether the keyspace.table has the input partition key denied         |
+----------------------------------------------------------------------------+---------------------------------------------------------------------------------+
