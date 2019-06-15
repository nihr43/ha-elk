# ha-elk
A highly available elasticsearch and kibana installation

Thanks to its internal routing and access via REST, elasticsearch clusters work very nicely behind haproxy in round-robin.  Recently I discovered that kibana actually stores most of its configuration in a ".kibana" index within elasticsearch, making it very easy to load-balance kibana instances as well.  So, here I share my design for a rather robust and elegant elasticsearch - kibana cluster.

## Rational

Why not docker?
 - I need somewhere to pour `all the data`.  Including the kubernetes data.  Its sort of a chicken-and-egg problem.  If elk is bootstrapped in a container management system, then where send those data?  Also, persisted state in kubernetes/swarm is hard.

Why FreeBSD?
 - `/` on zfs with ARC compression. top: `13G Compressed, 31G Uncompressed, 2.35:1 Ratio`
 - For somewhere I plan to "pour all the data", I want it to be essentially bomb-proof.  Modern FreeBSD gives you that, as long a you treat zfs well.

## Design

For this example I have two physical servers, `8f14-jmm` and `11e9-jmm`.  Each has failover network paths, failover power supplies, and raid50 zroot pools on cheap sata ssds.

Each server is running a number of jails created with my `jm` tool.  These jails have full network stacks bridged to the host, and are managed using ansible.

```text
+----------------------------------+
| 0def-switch                      |---+----+
+----------------------------------+   |    |
+----------------------------------+   |    |
| 1f03-switch                      |-+-|--+ |
+----------------------------------+ | |  | |
                                     | |  | |
+----------------------------------+ | |  | |
| 8f14-jmm                         | | |  | |
|                              +-+ | | |  | |
| 11e9-haproxy-kibana        - |>| | | |  | |
| 8f15-kibana                - |<|-|-+ |  | |
| 9fbb-haproxy-elasticsearch - |>|-|---+  | |
| d12c-elasticsearch         - |<| |      | |
|                              +-+ |      | |
+----------------------------------+      | |
                                          | |
+----------------------------------+      | |
| 11e9-jmm                         |      | |
|                              +-+ |      | |
| ba5a-haproxy-kibana        - |<| |      | |
| 11e9-kibana                - |>|-|------+ |
| 8f15-haproxy-elasticsearch - |<|-|--------+
| 39b6-elasticsearch         - |>| |
|                              +-+ |
+----------------------------------+
```

Traffic flow:
 - traffic arrives at kibana haproxy master via virtual CARP ip
 - haproxy forwards traffic to a random kibana node
 - kibana looks for database at elasticsearch CARP address
 - traffic arrives at elasticsearch haproxy master via virtual CARP ip
 - haproxy forwards traffic to a random elasticsearch cluster node
 - elasticsearch internally routes traffic to an appropriate shard
