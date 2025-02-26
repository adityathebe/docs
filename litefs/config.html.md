---
title: LiteFS Config Reference
layout: docs
sitemap: false
nav: litefs
toc: true
---

LiteFS is primarily configured via a `litefs.yml` config file. This file can
be in the current directory, the home directory, or at `/etc/litefs.yml`. You
can also pass in a path using a command line flag:

```sh
litefs -config /path/to/litefs.yml
```

## Directories

LiteFS works by mounting a file system to a _mount directory_ and storing the
underlying data in a _data directory_. The mount directory is the path that
your application will use to interact with LiteFS.

The mount directory is a required field in the config file. However, if the data
directory is omitted, LiteFS will use a hidden directory based on the name of
the mount directory. For example, a mount directory of `/path/to/mnt` will have
a default data directory of `/path/to/.mnt`

```yml
# Required. Path used to access LiteFS from your application.
mount-dir: "/path/to/mnt"

# Optional. Path to store underlying data.
data-dir: "/path/to/data"
```

## Supervisor / Exec

LiteFS provides the option to run as a supervisor to another process. Typically,
this would be the command you run for your application. This is useful so that
LiteFS can mount itself and connect to the cluster before starting your
application. It will pass signals to your application and it will automatically
shutdown when your application shuts down.

```yml
exec: "myapp -addr :8080"
```


## Leadership Settings

By default, any node in the LiteFS cluster can be a "candidate" which means that
they are eligible to become the leader. However, if you want to restrict
candidancy to only a subset of nodes, you can mark non-candidate nodes in
their config:

```yml
# The candidate flag specifies whether the node
# can become the primary.
candidate: false
```


## Debug

LiteFS will generate regular log entries to STDOUT so users can verify that
nodes are connecting and replicating. Additionally, you can specify the `debug`
setting to also add verbose FUSE logging. This may be helpful when debugging or
reporting an issue to the project.

```yml
# The debug flag enables debug logging of all FUSE API calls.
debug: true
```

## Retention

Each transaction performed in LiteFS will generate an LTX file. These files are
replicated to other nodes in the cluster to propagate changes. Eventually, these
files could fill the local disk if they're not removed so LiteFS runs retention
enforcement on regular intervals.

By default, LiteFS will remove any LTX files that are more than 1 minute old and
it will perform this retention check every minute. You can update these settings
in the `retention` section of the config:

```yml
retention:
  # The amount of time to keep LTX files.
  # Latest LTX file is always kept.
  duration: "60s"

  # The frequency with which to check for LTX files to delete.
  monitor-interval: "60s"
```


## HTTP Server

LiteFS communicates with other nodes over HTTP and this can be configured in the
`http` section of the config. Currently, there is only an `addr` field to
specify the bind address, which defaults to `":20202"`


```yml
http:
  # Specifies the bind address of the HTTP API server.
  addr: ":20202"
```


## Lease Management

Leadership is managed by using an external lease system. This can be done using
either Consul or by specifying a single, static primary node.

### Consul Leasing

Consul is the recommended lease backend as it allows leadership to change when
the primary node stops. The `url` and `advertise-url` are required so that the
node can connect to Consul and broadcast its LiteFS API URL.

The `hostname` is the name that will be shown to replicas in the `.primary` file
so it should be set to a hostport that is accessible to other nodes in the
cluster. This is set to the value of [`hostname(1)`](https://linux.die.net/man/1/hostname).

The `key` specifies the name to store the primary data under in Consul. It can
be useful to change this if you are running multiple LiteFS clusters with the
same Consul instance.

The `ttl` and `lock-delay` specify how long a lock exists before expiration and
how long until a new lock can be obtained after expiration. These only apply to
primary nodes when they are uncleanly shutdown. If a node is properly shutdown
via `SIGINT` then it will remove its lease and another node can obtain it
immediately.

```yml
# A Consul server provides leader election and ensures that the
# responsibility of the primary node can be moved in the event
# of a deployment or a failure.
consul:
  # Required. The base URL of the Consul server.
  url: "http://localhost:8500"

  # Required. The URL that litefs is accessible on.
  advertise-url: "http://localhost:20202"

  # Sets the hostname that other nodes will use to reference node.
  # Automatically assigned based on hostname(1) if not set.
  hostname: "localhost"

  # The key used for obtaining a lease by the primary.
  # This must be unique for each cluster of LiteFS servers
  key: "litefs/primary"

  # Length of time before a lease expires.
  ttl: "10s"

  # Length of time after the lease expires before a
  # candidate can become leader.
  lock-delay: "5s"
```


### Static Leasing

If you do not wish to run a separate Consul instance and you can tolerate
downtime of your primary node, you can run LiteFS with a static lease. A static
lease means that only a single, fixed node will ever be the primary.

To set this up, set the `static.hostname` and `static.advertise-url` to the
values for the primary node. This should be done on the config for all nodes in
the cluster.

Then, set the `static.primary` field to `true` on the primary node and `false`
on all other nodes.


```yml
# Static leadership can be used instead of Consul if only one
# node should ever be the primary. Only one node in the cluster
# can be marked as the "primary".
static:
  # Specifies that the current node is the primary.
  primary: true

  # Required. Hostname of the primary node.
  hostname: "localhost"

  # Required. The API URL of the primary node.
  advertise-url: "http://localhost:20202"
```


