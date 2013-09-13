bootstrap
=========

The `bootstrap` application is a simple, yet poweful application to bootstrap an
Erlang cluster without having to know the exact nodenames or hostnames in
advance. It automates the process of dynamically populating the cluster with new
nodes.

* [Code](http://github.com/schlagert/bootstrap)
* [EDoc](http://schlagert.github.com/bootstrap)
* [![Build Status](https://travis-ci.org/schlagert/bootstrap.png?branch=master)](https://travis-ci.org/schlagert/bootstrap)

Features
--------

* Automatically discover and connect to nodes in an Erlang cluster.
* Utilize UDP broadcast or multicast for node discovery.
* Suppport multiple nodes per host.
* Support creation of complex topologies using `hidden` connections.
* Provide node connection/disconnection notification system.

How it works
------------

TODO

Configuration
-------------

The `bootstrap` application already comes with sensible defaults (except for the
regex used to decide whether to connect to a certain node or not). However,
many things can be customized if desired. For this purpose the following
configuration options are available and can be configured in the application
environment:

* `{connect_regex, Regex :: string()}`

  Specifies a regular expression to be compiled using `re:compile/1`. A
  bootstrap instance will only connect to a node if its name matches this
  expression. Furthermore, transitive connections implicitly made by `global`
  (in visible mode) will only be reported if they match this expression. Default
  is `".*"`.

* `{connect_mode, visible | hidden}`

  Specifies the type of connections that will be established by this protocol
  instance. Visible connections will most probably pull in transitive
  connections to other nodes. If a specialized, custom topology is desired this
  value should be set to `hidden`. However, __global name registration is
  disfunctional__ over `hidden` connections. Default is `visible`.

* `{min_connections, non_neg_integer() | infinity}`

  Specifies the minimum number of __matching__ connections. A node will actively
  try to get connections to matching nodes up to this number. Further
  connections may be established, but the active part of the discovery will be
  stopped when reaching this number. Active discovery will start again when the
  number of connections drops beneath this value. Default is `infinity`.

* `{primary_port, inet:port_number()}`

  Specifies the main network port broadcast or multicast packets will be sent
  over. This value must be equal for all `bootstrap` instances on all nodes.
  Default is `50337`.

* `{secondary_ports, [inet:port_number()]}`

  Specified additional listen ports. This feature is necessary if the system
  involves more than one node per host, since there's no common way to listen
  on the same network port from multiple processes. For each additional node per
  host another network port is required. E.g. if a system has a host with three
  nodes this list must contain two entries (different to the `primary_port`
  value). Default is `[50338, 50339]`.

* `{protocol, broadcast | multicast}`

  Specifies the network protocol used for node discovery. Default is
  `broadcast`.

* `{ping_timeout, non_neg_integer()}`

  Specifies the rough time in milliseconds between two consecutive, active node
  discovery messages. Default is `10000`.

* `{multicast_ip, MulticastAddr :: inet:ip4_address()}`

  Specifies the multicast address to be used for node discovery when using
  the `multicast` protocol. Default is `{239, 192, 0, 1}`.

* `{multicast_ttl, non_neg_integer()}`

  Specifies the time-to-live (TTL) of outgoing multicast packets. When setting
  the TTL to `1` all multicast packets are limited to the local network. Default
  is `1`.

The default configuration can be used to form an unbounded, mesh-connected
Erlang cluster utilizing the UDP broadcast protocol. Please note that to be able
to connect to other Erlang nodes, these nodes __must have the same Erlang
cookie__ configured.

Notifications
-------------

If the you use `bootstrap` to automatically establish connections between nodes,
configuring and starting the application on all nodes is basically all you need.
However, some use cases may make it necessary to get notified whenever a
__matching__ connection is established or lost.

For this purpose the `bootstrap` application provides the `bootstrap` behaviour.
To get notifications about node actions the two functions `on_connected/2` and
`on_disconnected/3` must be implemented. The implementing handler can the be
managed using the functions provided in the `bootstrap` module. If you already
know the `gen_event` behaviour, this will be nothing new for you. All functions
except for `add_sup_handler/2` basically do the same as the `gen_event`
equivalents. The only difference between `add_handler/2` and `add_sup_handler/2`
is that the added handler will automatically be removed when the calling process
exits. No messages will be sent to the calling process.

`bootstrap` handlers will get initial notifications for all __matching__ nodes
that are currently connected.

For more information, please refer to the `edoc` of the `bootstrap` module.

Security
--------

The `bootstrap` application __is__ insecure. This means everyone with access to
the used broadcast/multicast domain can send, receive and read `bootstrap`
packets. __However__, to be able to connect to any of the gathered nodenames an
attacker must know the used Erlang cookie. The `bootstrap` application will
never transmit cookies over the wire in any form.

Examples
--------

### Mesh Topology

This is the basic setup you get when connecting to other nodes. Every node is
connected with every other node in the cluster. The view for `bootstrap`
handlers does not differ from the `net_kernel` view:

<img src="http://schlagert.github.com/bootstrap/mesh.svg" alt="Mesh Topology with visible connections." />

The `sys.config` configuration to build a topology like this would look like the
following (on all nodes):
```erlang
[{bootstrap, [{connect_regex, ".*"}, {min_connections, 1}]}].
```

### Star Topology

TODO

<img src="http://schlagert.github.com/bootstrap/star.svg" alt="Star Topology with visible and hidden connections." />

The `sys.config` configuration to build a topology like this would look like the
following:

* with `visible` connections (left example)

Master node:
```erlang
[{bootstrap, [{connect_regex, "slave@.*"}, {min_connections, 1}]}].

Slaves nodes:
[{bootstrap, [{connect_regex, "master@.*"}, {min_connections, 1}]}].
```

* with `hidden` connections (right example)

Master node:
```erlang
[{bootstrap, [{connect_regex, "slave@.*"}, {connect_mode, hidden}, {min_connections, 1}]}].

Slaves nodes:
[{bootstrap, [{connect_regex, "master@.*"}, {connect_mode, hidden}, {min_connections, 1}]}].
```

### Tree Topology

<img src="http://schlagert.github.com/bootstrap/tree.svg" alt="Tree Topology with hidden connections." />

`hidden` connections

TODO

History
-------

### Version 0.0.1

* Regular expression based node matching
* UDP broadcast support
* UDP multicast support
* Multi-node-per-host support
* Avoid duplicate broadcasts to minimize network usage
* Behaviour-based notification system
