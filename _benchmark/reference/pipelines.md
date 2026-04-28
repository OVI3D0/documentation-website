---
layout: default
title: Pipelines
nav_order: 35
parent: Reference
---

# Pipelines

A pipeline defines the steps OpenSearch Benchmark performs before, during, and after a benchmark run. The pipeline controls whether OpenSearch Benchmark provisions its own cluster or connects to an existing one.

Select a pipeline using the `--pipeline` flag:

```bash
opensearch-benchmark run --pipeline=benchmark-only --workload=geonames --target-hosts=localhost:9200
```

## Available pipelines

### benchmark-only

The **default** pipeline. Assumes an OpenSearch cluster is already running and accessible. OpenSearch Benchmark connects to it, runs the workload, and publishes results.

Use this pipeline when:
- You have an existing cluster you want to benchmark
- You're benchmarking a managed service (Amazon OpenSearch Service, etc.)
- You want full control over cluster configuration

```bash
opensearch-benchmark run \
  --pipeline=benchmark-only \
  --workload=geonames \
  --target-hosts=my-cluster:9200
```

### from-distribution

Downloads an official OpenSearch distribution, provisions a local single-node cluster, runs the workload, and shuts down the cluster afterward.

Use this pipeline when:
- You want to benchmark a specific OpenSearch version without manual setup
- You're comparing performance across OpenSearch versions
- You need a quick, reproducible benchmark environment

```bash
opensearch-benchmark run \
  --pipeline=from-distribution \
  --distribution-version=2.19.1 \
  --workload=geonames
```

OpenSearch Benchmark downloads the distribution to `~/.benchmark/benchmarks/distributions/` and caches it for future runs. The cluster starts on a random port (shown in the output) and is automatically stopped when the benchmark completes.

**Requirements:**
- Java Development Kit (JDK) installed. OpenSearch Benchmark searches for JDK versions 17, 16, 15, 14, 13, 12, 11, or 8 using environment variables like `JAVA17_HOME` or `JAVA_HOME`.
- Sufficient disk space for the OpenSearch distribution (~500MB) and benchmark data.

### from-sources

Builds OpenSearch from source code, provisions the built cluster, runs the workload, and shuts down.

Use this pipeline when:
- You're developing OpenSearch and want to benchmark your changes
- You need to benchmark a pre-release build
- You want to test custom plugins compiled from source

```bash
opensearch-benchmark run \
  --pipeline=from-sources \
  --revision=latest \
  --workload=geonames
```

**Requirements:**
- Git
- JDK 17+
- Gradle
- The OpenSearch source repository (configured in `benchmark.ini` under `[source]`)

### docker

Uses Docker to provision an OpenSearch cluster for benchmarking.

```bash
opensearch-benchmark run \
  --pipeline=docker \
  --distribution-version=2.19.1 \
  --workload=geonames
```

**Requirements:**
- Docker installed and running

## Pipeline comparison

| Pipeline | Provisions cluster? | Requires JDK? | Best for |
|----------|-------------------|---------------|----------|
| `benchmark-only` | No | No | Existing clusters, managed services |
| `from-distribution` | Yes (download) | Yes | Version comparison, quick tests |
| `from-sources` | Yes (build) | Yes | Development, pre-release testing |
| `docker` | Yes (container) | No | Isolated, reproducible environments |

## How pipelines work internally

Each pipeline follows this sequence:

1. **Provision** (skip for `benchmark-only`): Download/build/pull OpenSearch and start the cluster
2. **Wait for cluster**: Poll the cluster health endpoint until it responds
3. **Retrieve cluster info**: Get the cluster version and metadata
4. **Prepare workload**: Load the workload definition, download data files if needed
5. **Execute benchmark**: Run the test procedure's schedule (ingest, search, etc.)
6. **Publish results**: Store metrics in the configured datastore
7. **Tear down** (skip for `benchmark-only`): Stop and clean up the provisioned cluster

The provisioning step is the only difference between pipelines. Steps 2--7 are identical regardless of which pipeline you use.

## Common options

The following options work with all pipelines:

| Flag | Description |
|------|-------------|
| `--workload` | The workload to run |
| `--test-procedure` | Which test procedure within the workload to execute |
| `--workload-params` | Parameters to pass to the workload |
| `--target-hosts` | The cluster endpoint(s) to benchmark (required for `benchmark-only`) |
| `--test-mode` | Run with minimal data for quick validation |
| `--kill-running-processes` | Kill any existing OpenSearch Benchmark processes before starting |

For the full list of command flags, see [Command flags]({{site.url}}{{site.baseurl}}/benchmark/reference/commands/command-flags/).
