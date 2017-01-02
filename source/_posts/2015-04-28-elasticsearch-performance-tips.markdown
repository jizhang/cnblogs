
---
layout: post
title: "ElasticSearch Performance Tips"
date: 2015-04-28 23:08
categories: [Programming]
comments: true
tags: [elasticsearch, english]
---

Recently we're using ElasticSearch as a data backend of our recommendation API, to serve both offline and online computed data to users. Thanks to ElasticSearch's rich and out-of-the-box functionality, it doesn't take much trouble to setup the cluster. However, we still encounter some misuse and unwise configurations. So here's a list of ElasticSearch performance tips that we learned from practice.

## Tip 1 Set Num-of-shards to Num-of-nodes

[Shard][1] is the foundation of ElasticSearch's distribution capability. Every index is splitted into several shards (default 5) and are distributed across cluster nodes. But this capability does not come free. Since data being queried reside in all shards (this behaviour can be changed by [routing][2]), ElasticSearch has to run this query on every shard, fetch the result, and merge them, like a map-reduce process. So if there're too many shards, more than the number of cluter nodes, the query will be executed more than once on the same node, and it'll also impact the merge phase. On the other hand, too few shards will also reduce the performance, for not all nodes are being utilized.

Shards have two roles, primary shard and replica shard. Replica shard serves as a backup to the primary shard. When primary goes down, the replica takes its job. It also helps improving the search and get performance, for these requests can be executed on either primary or replica shard.

Shards can be visualized by [elasticsearch-head][1] plugin:

![](/images/elasticsearch/shards-head.png)

The `cu_docs` index has two shards `0` and `1`, with `number_of_replicas` set to 1. Primary shard `0` (bold bordered) resides in server `Leon`, and its replica in `Pris`. They are green becuase all primary shards have enough repicas sitting in different servers, so the cluster is healthy.

Since `number_of_shards` of an index cannot be changed after creation (while `number_of_replicas` can), one should choose this config wisely. Here are some suggestions:

1. How many nodes do you have, now and future? If you're sure you'll only have 3 nodes, set number of shards to 2 and replicas to 1, so there'll be 4 shards across 3 nodes. If you'll add some servers in the future, you can set number of shards to 3, so when the cluster grows to 5 nodes, there'll be 6 distributed shards.
2. How big is your index? If it's small, one shard with one replica will due.
3. How is the read and write frequency, respectively? If it's search heavy, setup more relicas. 

<!-- more -->

## Tip 2 Tuning Memory Usage

ElasticSearch and its backend [Lucene](http://lucene.apache.org/) are both Java application. There're various memory tuning settings related to heap and native memory.

### Set Max Heap Size to Half of Total Memory

Generally speaking, more heap memory leads to better performance. But in ElasticSearch's case, Lucene also requires a lot of native memory (or off-heap memory), to store index segments and provide fast search performance. But it does not load the files by itself. Instead, it relies on the operating system to cache the segement files in memory.

Say we have 16G memory and set -Xmx to 8G, it doesn't mean the remaining 8G is wasted. Except for the memory OS preserves for itself, it will cache the frequently accessed disk files in memory automatically, which results in a huge performance gain.

Do not set heap size over 32G though, even you have more than 64G memory. The reason is described in [this link][4].

Also, you should probably set -Xms to 8G as well, to avoid the overhead of heap memory growth.

### Disable Swapping

Swapping is a way to move unused program code and data to disk so as to provide more space for running applications and file caching. It also provides a buffer for the system to recover from memory exhaustion. But for critical application like ElasticSearch, being swapped is definitely a performance killer.

There're several ways to disable swapping, and our choice is setting `bootstrap.mlockall` to true. This tells ElasticSearch to lock its memory space in RAM so that OS will not swap it out. One can confirm this setting via `http://localhost:9200/_nodes/process?pretty`.

If ElasticSearch is not started as root (and it probably shouldn't), this setting may not take effect. For Ubuntu server, one needs to add `<user> hard memlock unlimited` to `/etc/security/limits.conf`, and run `ulimit -l unlimited` before starting ElasticSearch process.

### Increase `mmap` Counts

ElasticSearch uses memory mapped files, and the default `mmap` counts is low. Add `vm.max_map_count=262144` to `/etc/sysctl.conf`, run `sysctl -p /etc/sysctl.conf` as root, and then restart ElasticSearch.

## Tip 3 Setup a Cluster with Unicast

ElasticSearch has two options to form a cluster, multicast and unicast. The former is suitable when you have a large group of servers and a well configured network. But we found unicast more concise and less error-prone.

Here's an example of using unicast:

```
node.name: "NODE-1"
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: ["node-1.example.com", "node-2.example.com", "node-3.example.com"]
discovery.zen.minimum_master_nodes: 2
```

The `discovery.zen.minimum_master_nodes` setting is a way to prevent split-brain symptom, i.e. more than one node thinks itself the master of the cluster. And for this setting to work, you should have an odd number of nodes, and set this config to `ceil(num_of_nodes / 2)`. In the above cluster, you can lose at most one node. It's much like a quorum in [Zookeeper](http://zookeeper.apache.org).

## Tip 4 Disable Unnecessary Features

ElasticSearch is a full-featured search engine, but you should always tailor it to your own needs. Here's a brief list:

* Use corrent index type. There're `index`, `not_analyzed`, and `no`. If you don't need to search the field, set it to `no`; if you only search for full match, use `not_analyzed`.
* For search-only fields, set `store` to false.
* Disable `_all` field, if you always know which field to search.
* Disable `_source` fields, if documents are big and you don't need the update capability.
* If you have a document key, set this field in `_id` - `path`, instead of index the field twice.
* Set `index.refresh_interval` to a larger number (default 1s), if you don't need near-realtime search. It's also an important option in bulk-load operation described below.

## Tip 5 Use Bulk Operations

[Bulk is cheaper][5]

* Bulk Read
    * Use [Multi Get][6] to retrieve multiple documents by a list of ids. 
    * Use [Scroll][7] to search a large number of documents.
    * Use [MultiSearch api][8] to run search requests in parallel. 
* Bulk Write
    * Use [Bulk API][9] to index, update, delete multiple documents.
    * Alter [index aliases][10] simultaneously.
* Bulk Load: when initially building a large index, do the following,
    * Set `number_of_relicas` to 0, so no relicas will be created;
    * Set `index.refresh_interval` to -1, disabling nrt search;
    * Bulk build the documents;
    * Call `optimize` on the index, so newly built docs are available for search;
    * Reset replicas and refresh interval, let ES cluster recover to green.

## Miscellaneous

* File descriptors: system default is too small for ES, set it to 64K will be OK. If `ulimit -n 64000` does not work, you need to add `<user> hard nofile 64000` to `/etc/security/limits.conf`, just like the `memlock` setting mentioned above.
* When using ES client library, it will create a lot of worker threads according to the number of processors. Sometimes it's not necessary. This behaviour can be changed by setting `processors` to a lower value like 2:

```scala
val settings = ImmutableSettings.settingsBuilder()
    .put("cluster.name", "elasticsearch")
    .put("processors", 2)
    .build()
val uri = ElasticsearchClientUri("elasticsearch://127.0.0.1:9300")
ElasticClient.remote(settings, uri)
```

## References

* https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html
* https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
* http://cpratt.co/how-many-shards-should-elasticsearch-indexes-have/
* https://www.elastic.co/blog/performance-considerations-elasticsearch-indexing
* https://www.loggly.com/blog/nine-tips-configuring-elasticsearch-for-high-performance/

[1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/glossary.html#glossary-shard
[2]: https://www.elastic.co/guide/en/elasticsearch/reference/current/glossary.html#glossary-routing
[3]: http://mobz.github.io/elasticsearch-head/
[4]: https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html#compressed_oops
[5]: https://www.elastic.co/guide/en/elasticsearch/guide/current/bulk.html
[6]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-multi-get.html
[7]: https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-scroll.html
[8]: https://www.elastic.co/guide/en/elasticsearch/client/java-api/1.4/msearch.html
[9]: https://www.elastic.co/guide/en/elasticsearch/client/java-api/1.4/bulk.html
[10]:https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html
