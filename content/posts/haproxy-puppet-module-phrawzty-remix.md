+++
title = "HAProxy Puppet module (phrawzty remix)"
date = 2013-03-18
+++

As part of a big [Logstash](http://www.logstash.net/) project at Mozilla (more on that to come), I was looking for an [HAProxy](http://haproxy.1wt.eu/) module for Puppet, stumbling across the official [Puppetlabs module](https://forge.puppetlabs.com/puppetlabs/haproxy) in the process.  I'm told that this module works fairly well, with the caveat that it sometimes outputs poorly-formatted configuration files (due to a manifestly buggy implementation of [concat](https://github.com/ripienaar/puppet-concat)).  Furthermore, the module more or less requires [storeconfigs](https://www.google.fr/search?q=puppet+storeconfigs), which we do not use in our primary Puppet system.
Long story short, while I never ended up using HAProxy as part of the project, I did *remix* the official module to solve both of the aforementioned issues.  From the README :

This module is based on Puppetlabs' official HAProxy module; however, it has been "remixed" for use at Mozilla. There are two major areas where the original module has been changed :

* Storeconfigs, while present, are no longer required.
* The "listen" stanza format has been abandoned in favour of a frontend / backend style configuration.

A very simple configuration to proxy unrelated Redis nodes :

```bash
  class { 'haproxy': }

  haproxy::frontend { 'in_redis':
    ipaddress       => $::ipaddress,
    ports           => '6379',
    default_backend => 'out_redis',
    options         => { 'balance' => 'roundrobin' }
  }

  haproxy::backend { 'out_redis':
    listening_service => 'redis',
    server_names      => ['node01', 'node02'],
    ipaddresses       => ['node01.redis.server.foo', 'node02.redis.server.foo'],
    ports             => '6379',
    options           => 'check'
  }
```

If that sounds interesting to you, the module is available on my [puppetlabs-haproxy repo](https://github.com/phrawzty/puppetlabs-haproxy) on Github. Pull requests welcome !