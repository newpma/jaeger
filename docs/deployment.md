# Deployment

The main Jaeger backend components are released as Docker images on Docker Hub:

Component             | Repository
--------------------- | ---
**jaeger-agent**      | [hub.docker.com/r/jaegertracing/jaeger-agent/](https://hub.docker.com/r/jaegertracing/jaeger-agent/)
**jaeger-collector**  | [hub.docker.com/r/jaegertracing/jaeger-collector/](https://hub.docker.com/r/jaegertracing/jaeger-collector/)
**jaeger-query**      | [hub.docker.com/r/jaegertracing/jaeger-query/](https://hub.docker.com/r/jaegertracing/jaeger-query/)

There are orchestration templates for running Jaeger with:

  * Kubernetes: [github.com/jaegertracing/jaeger-kubernetes](https://github.com/jaegertracing/jaeger-kubernetes),
  * OpenShift: [github.com/jaegertracing/jaeger-openshift](https://github.com/jaegertracing/jaeger-openshift).

As well as a [docker-compose.yml](./../cmd/jaeger-stack) for your convenience to start up the whole stack including the query interface, a Cassandra database, an agent and the collector on one machine. The compose-file will kick start the database (Cassandra) and a container that establishes the database-structure as soon as Cassandra is available. After Cassandras successful configuration, the collector and the query interface will become stable as well (while the schema setup is in progress you may see the collector and the query restart until eventually all their needs are satisfied).

## Agent

Jaeger client libraries expect **jaeger-agent** process to run locally on each host.
The agent exposes the following ports:

Port | Protocol | Function
---- | -------  | ---
5775 | UDP      | accept zipkin.thrift over compact thrift protocol
6831 | UDP      | accept jaeger.thrift over compact thrift protocol
6832 | UDP      | accept jaeger.thrift over binary thrift protocol
5778 | HTTP     | serve configs, sampling strategies

### Discovery System Integration

The agents can connect point to point to a single collector address, which could be
load balanced by another infrastructure component (e.g. DNS) across multilpe collectors.
The agent can also be configured with a static list of collector addresses.

In the future we will support different service discovery systems to dynamically load balance
across several collectors ([issue 213](https://github.com/uber/jaeger/issues/213)).

## Collectors

Many instances of **jaeger-collector** can be run in parallel.
Collectors require almost no configuration, except for the location of Cassandra cluster,
via `-cassandra.keyspace` and `-cassandra.servers` options. To see all command line options run

```
go run ./cmd/collector/main.go -h
```

or, if you don't have the source code

```
docker run -it --rm jaegertracing/jaeger-collector /go/bin/collector-linux -h
```

At default settings the collector exposes the following ports: 

Port  | Protocol | Function
----- | -------  | ---
14267 | TChannel | used by **jaeger-agent** to send spans in jaeger.thrift format
14268 | HTTP     | can accept spans directly from clients in Jaeger or Zipkin Thrift 


## Storage Backend

Collectors require a persistent storage backend. Cassandra 3.x is the primary supported storage.
There is ongoing work to add support for Elasticsearch, MySQL, and ScyllaDB.

### Cassandra

A script is provided to initialize Cassandra keyspace and schema
using Cassandra's interactive shell [`cqlsh`][cqlsh]:

```sh
MODE=test sh ./plugin/storage/cassandra/schema/create.sh | cqlsh
```

For production deployment, pass `MODE=prod DATACENTER={datacenter}` arguments to the script,
where `{datacenter}` is the name used in the Cassandra configuration / network topology.

The script also allows overriding TTL, keyspace name, replication factor, etc.
Run the script without arguments to see the full list of recognized parameters.

## Query Service & UI

**jaeger-query** serves the API endpoints and a React/Javascript UI.
The service is stateless and is typically run behind a load balancer, e.g. nginx.

At default settings the query service exposes the following port(s): 

Port  | Protocol | Function
----- | -------  | ---
16686 | HTTP     | **/api/*** endpoints and Jaeger UI at **/**

TODO: Swagger and GraphQL API ([issue 158](https://github.com/uber/jaeger/issues/158)).

## Aggregation Jobs for Service Dependencies

At the moment this is work in progress. We're working on a post-processing data pipeline
that will include aggregating data to present service dependency diagram.


[cqlsh]: http://cassandra.apache.org/doc/latest/tools/cqlsh.html
