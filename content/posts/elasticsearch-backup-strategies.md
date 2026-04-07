+++
title = "Elasticsearch backup strategies"
date = 2011-11-22
+++

**Update**: This is an old blog post and is no longer relevant as of version 1.x of Elasticsearch. Now we can just use the [snapshot](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-snapshots.html) feature.
Hello again! Today we're going to talk about backup strategies for [Elasticsearch](http://elasticsearch.org). One popular way to make backups of ES requires the use of separate ES node, while another relies entirely on the underlying file system of a given set of ES nodes.
The ES-based approach:

* Bring up an independent (receiving) ES node on a machine that has network access to the actual ES cluster.
* Trigger a script to perform a full index import *from* the ES cluster *to* the receiving node.
* Since the receiving node is unique, every shard will be represented on said node.
* Shutdown the receiving node.
* Preserve the `/data/` directory from the receiving node.

The file system-based approach:

* Identify a *quorum* of nodes in the ES cluster.
* Quorum is necessary in order to ensure that all of the shards are represented.
* Trigger a script that will preserve the `/data/` directory of each selected node.

At first glance the file system-based approach appears simpler - and it is - but it comes with some drawbacks, notably the fact that coherency is *impossible to guarantee* due to the amount of time required to preserve `/data/` on each node. In other words, if data changes on node between the start and end times of the preservation mechanism, those changes may or may not be backed up. Furthermore, from an operational perspective, restoring nodes from individual shards may be problematic.
The ES-based approach does not have the coherency problem; however, beyond the fact that it is more complex to implement and maintain, it is also more costly in terms of service delivery. The actual import process itself requires a large number of requests to be made to the cluster, and the resulting resource consumption on both the cluster nodes as well as the receiving node are non-trivial. On the other hand, having a single, coherent representation of every shard in one place may pay dividends during a restoration scenario.
As is often the case, there is no one solution that is going to work for everybody all of the time - different environments have different needs, which call for different answers.  That said, if your *primary* goal is a consistent, coherent, and complete backup that can be easily restored when necessary (and overhead be damned!), then the ES-based approach is clearly the superior of the two.

### import it !

Regarding the ES-based approach, it may be helpful to take a look at a simple import script as an example.  How about a quick and dirty Perl script (straight from [the docs)](https://github.com/clintongormley/ElasticSearch.pm) ?

```bash
use ElasticSearch;

my $local = ElasticSearch->new(
    servers => 'localhost:9200'
);
my $remote = ElasticSearch->new(
    servers    => 'cluster_member:9200',
    no_refresh => 1
);

my $source = $remote->scrolled_search(
    index => 'content',
    search_type => 'scan',
    scroll      => '5m'
);
$local->reindex(source=>$source);
```

You'll want to replace the relevant elements with something sane for your environment, of course.
As for preserving the resulting /data/ directory (in either method), I will leave that as an exercise to the reader, since there are simply too many equally relevant ways to go about it.  It's worth noting that the import method doesn't need to be complex *at all* - in fact, it really shouldn't be, since complex backup schemes tend to have too many chances for failure than is necessary.
Happy indexing!