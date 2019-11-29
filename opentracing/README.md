# OpenTracing

### What is Distributed Tracing?

Distributed tracing is a method used to profile and monitor applications, especially those built using a microservices architecture. Distributed tracing helps pinpoint where failures occur and what causes poor performance.

## How OpenTracing Fits Into This?

The OpenTracing API provides a standard, vendor-neutral framework for instrumentation. This means that if a developer wants to try out a different distributed tracing system, then instead of repeating the whole instrumentation process for the new distributed tracing system, the developer can simply change the configuration of the Tracer.

Here are some basic terminologies of Opentracing:

### Span 
  It represents a logical unit of work that has an operation name, the start time of the operation, and the duration.

### Trace
   A Trace tells the story of a transaction or workflow as it propagates through a distributed system. It is simply a set of spans sharing a TraceID. Each component in a distributed system contributes its own span.
    
## Jaeger
Jaeger, inspired by Dapper and OpenZipkin, is a distributed tracing system released as open source by Uber Technologies. It is used for monitoring and troubleshooting microservices-based distributed systems, including:
- Distributed transaction monitoring
- Performance and latency optimization
- Root cause analysis
- Service dependency analysis
- Distributed context propagation


### Major Components of Jaeger

- Jaeger Client Libraries:
  Jaeger clients are language-specific implementations of the OpenTracing API.
- Agent: 
  The Jaeger agent is a network daemon that listens for spans sent over UDP, which it batches and sends to the collector. It is designed to be deployed to all hosts as an infrastructure component. The agent abstracts the routing and discovery of the collectors away from the client.
- Collector:
  The Jaeger collector receives traces from Jaeger agents and runs them through a processing pipeline. Currently, the pipeline validates traces, indexes them, performs transformations, and finally, stores them. Jaeger’s storage is a pluggable component which currently supports Cassandra, Elasticsearch, and Kafka.
- Query:
  Query is a service that retrieves traces from storage and hosts a UI to display them.
- Ingester: 
  Ingester is a service that reads from Kafka topic and writes to another storage backend (Cassandra, Elasticsearch).
