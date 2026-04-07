+++
title = "Improvements in Cassandra 1.0, briefly stated"
date = 2011-11-03
+++

Datastax recently announced the availability of Cassandra 1.0 (stable), and along with that announcement, they made a series of blog posts ([1](http://www.datastax.com/dev/blog/whats-new-in-cassandra-1-0-compression), [2](http://www.datastax.com/dev/blog/whats-new-in-cassandra-1-0-improved-memory-and-disk-space-management), [3](http://www.datastax.com/dev/blog/leveled-compaction-in-apache-cassandra), [4](http://www.datastax.com/dev/blog/whats-new-in-cassandra-1-0-performance), [5](http://www.datastax.com/dev/blog/whats-new-in-cassandra-1-0-windows-service-new-cql-clients-and-more)) about many of the great new features and improvements that the current version brings to the table.
For those of you looking for an executive summary of those posts, you're in luck, cause I've got your back on this one.

* New multi-layer approach to compression that provides improvements to *both* write and (especially) read operations.
* Said compression strategy also yields potentially significant disk space savings.
* Leverages the [JNA library](http://jna.java.net/) in order to provide in-memory caching; this procedure is optimised for garbage collection, resulting in a more efficient collection and a smaller overall footprint.
* A much improved compaction strategy results in less costly compaction runs, improving overall performance on each node.
* Fewer requests are made over the network, and said requests are smaller in size, improving overall performance across the cluster.

In short, 1.0 is a very significant, very important upgrade to 0.8 (et al.), and one which will likely bring it to the forefront of the hardcore big data / nosql scene at large.