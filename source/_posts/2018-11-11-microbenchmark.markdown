---
layout: post
title: "Microbenchmark"
date: 2018-11-11 13:58:58 -0800
comments: true
published: true
categories:
---

Microbenchmark is used to measure the performance of a small piece of code for the purpose of performance optimization. Writing a good microbenchmark is [hard](https://www.ibm.com/developerworks/library/j-benchmark1/index.html) and that's why we should use microbenchmark frameworks (e.g. [JMH](http://openjdk.java.net/projects/code-tools/jmh/) for Java and [Google Benchmark](https://github.com/google/benchmark) for C++) to help us. This post contains microbenchmarks that I think are interesting.

<!-- more -->

**Don't directly use the performance numbers in this post, do your own measurement!** Those numbers are highly dependent on the environment where those benchmarks are running.

``` c
BENCHMARK_DEFINE_F(BenchmarkFixture, ForLoopAssignmentBenchmark)(benchmark::State& state) {
  for (auto _ : state) {
    char c[2048];
    for (size_t i = 0; i < 2048; ++i) {
      c[i] = 'a';
    }
    benchmark::DoNotOptimize(c);
  }
}
BENCHMARK_REGISTER_F(BenchmarkFixture, ForLoopAssignmentBenchmark);

BENCHMARK_DEFINE_F(BenchmarkFixture, MemsetBenchmark)(benchmark::State& state) {
  for (auto _ : state) {
    char c[2048];
    memset(c, 'a', 2048);
    benchmark::DoNotOptimize(c);
  }
}
BENCHMARK_REGISTER_F(BenchmarkFixture, MemsetBenchmark);
```
```
Run on (12 X 2500 MHz CPU s)
------------------------------------------------------------------------------------
Benchmark                                              Time           CPU Iterations
------------------------------------------------------------------------------------
BenchmarkFixture/ForLoopAssignmentBenchmark         1013 ns       1013 ns     750073
BenchmarkFixture/MemsetBenchmark                      84 ns         84 ns    6209926
```
As we can see, memset is much faster than the for loop assignment in this case. Looking at the generated assembly code, memset uses `rep stos` instruction which can be the [reason](https://stackoverflow.com/questions/33480999/how-can-the-rep-stosb-instruction-execute-faster-than-the-equivalent-loop) why it's faster.

``` c
// https://shipilev.net/blog/2014/nanotrusting-nanotime/
inline uint64_t NanosecondsSinceEpoch() {
  timespec tp;
  clock_gettime(CLOCK_REALTIME, &tp);
  return tp.tv_sec * NS_PER_SEC + tp.tv_nsec;
}

BENCHMARK_DEFINE_F(BenchmarkFixture, NanoTimeLatencyBenchmark)(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(NanosecondsSinceEpoch());
  }
}
BENCHMARK_REGISTER_F(BenchmarkFixture, NanoTimeLatencyBenchmark);

BENCHMARK_DEFINE_F(BenchmarkFixture, NanoTimeGranularityBenchmark)(benchmark::State& state) {
  uint64_t cur_nano = 0;
  uint64_t last_nano = 0;
  for (auto _ : state) {
    do {
      cur_nano = NanosecondsSinceEpoch();
    } while (cur_nano == last_nano);
    last_nano = cur_nano;
  }
}
BENCHMARK_REGISTER_F(BenchmarkFixture, NanoTimeGranularityBenchmark);
```
```
Run on (12 X 2500 MHz CPU s)
------------------------------------------------------------------------------------
Benchmark                                              Time           CPU Iterations
------------------------------------------------------------------------------------
BenchmarkFixture/NanoTimeLatencyBenchmark             39 ns         39 ns   18361529
BenchmarkFixture/NanoTimeGranularityBenchmark         39 ns         39 ns   18073581
```
``` java
public class NanoTimeBenchmark {
  @State(Scope.Benchmark)
  public static class BenchmarkState {
    public long _lastNano;
    @Setup(Level.Iteration)
    public void setup() {
      _lastNano = 0;
    }
  }

  @Benchmark
  @Fork(1)
  @Threads(1)
  @Warmup(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
  @Measurement(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
  @BenchmarkMode(Mode.AverageTime)
  @OutputTimeUnit(TimeUnit.NANOSECONDS)
  public long nanoTimeLatency() {
    return System.nanoTime();
  }

  @Benchmark
  @Fork(1)
  @Threads(1)
  @Warmup(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
  @Measurement(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
  @BenchmarkMode(Mode.AverageTime)
  @OutputTimeUnit(TimeUnit.NANOSECONDS)
  public long nanoTimeGranularity(BenchmarkState benchmarkState) {
    long cur;
    do {
      cur = System.nanoTime();
    } while (cur == benchmarkState._lastNano);
    benchmarkState._lastNano = cur;
    return cur;
  }
}
```
```
Benchmark                              Mode  Cnt   Score   Error  Units
NanoTimeBenchmark.nanoTimeLatency      avgt   10  42.705 ± 0.302  ns/op
NanoTimeBenchmark.nanoTimeGranularity  avgt   10  43.875 ± 0.674  ns/op
```
This shows the overhead of getting time in both C++ and Java. They are not free!
