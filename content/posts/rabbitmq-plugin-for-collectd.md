+++
title = "RabbitMQ plugin for Collectd"
date = 2011-02-24
+++

Hello all,
I wrote a rudimentary RabbitMQ plugin for Collectd.  If that sounds interesting to you, feel free to take a look at my [GitHub](https://github.com/phrawzty/rabbitmq-collectd-plugin).  The plugin itself is written in Python and makes use of the Python plugin for Collectd.
It will accept four options from the Collectd plugin configuration :

Locations of binaries :

```bash
RmqcBin = /usr/sbin/rabbitmqctl
PmapBin = /usr/bin/pmap
PidofBin = /bin/pidof
```

Logging :

```bash
Verbose = false
```

It will attempt to gather the following information :

From « rabbitmqctl list\_queues » :

```bash
messages
memory
consumser
```

From « pmap » of « beam.smp » :

```bash
memory mapped
memory writeable/private (used)
memory shared
```

Props to Garret Heaton for inspiration and conceptual guidance from his « [redis-collectd-plugin](https://github.com/powdahound/redis-collectd-plugin) ».