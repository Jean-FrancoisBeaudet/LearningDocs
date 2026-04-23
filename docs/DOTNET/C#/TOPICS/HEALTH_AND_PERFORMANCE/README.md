# HEALTH_AND_PERFORMANCE

Topics extracted from a Senior .NET Performance Engineer job posting (`../plan.md`). Each file is a stub covering a discrete performance / diagnostics / runtime-health concept in C# / .NET.

## Profiling & diagnostics tools

- [dotMemory](profiling-dotmemory.md)
- [dotTrace](profiling-dottrace.md)
- [PerfView](profiling-perfview.md)
- [Visual Studio Diagnostic Tools](profiling-visual-studio-diagnostic-tools.md)
- [ETW (Event Tracing for Windows)](etw-event-tracing-for-windows.md)
- [EventPipe](eventpipe.md)
- [dotnet-trace](dotnet-trace.md)
- [dotnet-counters](dotnet-counters.md)
- [dotnet-dump](dotnet-dump.md)

## Memory management & GC

- [GC generations (gen0 / gen1 / gen2)](gc-generations-gen0-gen1-gen2.md)
- [Large Object Heap (LOH)](large-object-heap-loh.md)
- [LOH fragmentation](loh-fragmentation.md)
- [Pinned objects](pinned-objects.md)
- [Finalization and the finalizer queue](finalization-and-finalizer-queue.md)
- [Server vs Workstation GC](server-vs-workstation-gc.md)
- [DATAS (Dynamically Adapting To Application Sizes)](datas-dynamically-adapting-to-application-sizes.md)
- [Heap allocation patterns](heap-allocation-patterns.md)
- [Memory leaks in .NET](memory-leaks-in-dotnet.md)
- [GC pressure](gc-pressure.md)

## Performance anti-patterns & bottlenecks

- [Thread contention](thread-contention.md)
- [I/O saturation](io-saturation.md)
- [Allocations and boxing](allocations-and-boxing.md)
- [Closure captures](closure-captures.md)
- [LINQ performance misuse](linq-performance-misuse.md)
- [.NET performance anti-patterns](dotnet-performance-anti-patterns.md)

## High-throughput pipelines

- [Channel&lt;T&gt;](channel-of-t.md)
- [System.IO.Pipelines](system-io-pipelines.md)
- [Batch processing](batch-processing.md)
- [Streaming patterns](streaming-patterns.md)
- [Buffer management](buffer-management.md)
- [Backpressure handling](backpressure-handling.md)
- [async/await performance](async-await-performance.md)
- [Synchronization bottlenecks](synchronization-bottlenecks.md)

## Low-allocation patterns

- [Span&lt;T&gt;](span-of-t.md)
- [Memory&lt;T&gt;](memory-of-t.md)
- [ArrayPool&lt;T&gt;](arraypool-of-t.md)
- [Low-allocation patterns in modern .NET](low-allocation-patterns-modern-dotnet.md)

## Benchmarking & load testing

- [BenchmarkDotNet](benchmarkdotnet.md)
- [Load testing](load-testing.md)
- [Performance regression tracking](performance-regression-tracking.md)
- [Performance baselines and release gates](performance-baselines-and-release-gates.md)

## Database query performance

- [SQL index analysis](sql-index-analysis.md)
- [SQL query plan analysis](sql-query-plan-analysis.md)
- [Pagination strategies](pagination-strategies.md)
- [Aggregation efficiency](aggregation-efficiency.md)
- [Relational database query optimization](relational-database-query-optimization.md)
- [Document database query optimization](document-database-query-optimization.md)
- [RavenDB performance](ravendb-performance.md)
- [Bulk operations](bulk-operations.md)

## Distributed systems performance

- [Service mesh latency](service-mesh-latency.md)
- [Serialization overhead](serialization-overhead.md)
- [Connection pooling](connection-pooling.md)
- [Inter-service call optimization](inter-service-call-optimization.md)
