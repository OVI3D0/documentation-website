---
layout: default
title: Provisioning a cluster
nav_order: 8
parent: User guide
---

# Provisioning a cluster for benchmarking

OpenSearch Benchmark can either connect to an existing cluster or provision one automatically. This page covers the different ways to set up a cluster for benchmarking.

## Using an existing cluster (benchmark-only)

The simplest approach is to benchmark an already-running cluster using the `benchmark-only` pipeline (the default):

```bash
opensearch-benchmark run \
  --workload=geonames \
  --target-hosts=my-cluster:9200 \
  --pipeline=benchmark-only
```

This works with any OpenSearch cluster: self-managed, Docker, Amazon OpenSearch Service, or clusters deployed with infrastructure-as-code tools.

### Connecting with authentication

For clusters with security enabled:

```bash
opensearch-benchmark run \
  --workload=geonames \
  --target-hosts=https://my-cluster:9200 \
  --pipeline=benchmark-only \
  --client-options="use_ssl:true,verify_certs:true,basic_auth_user:admin,basic_auth_password:admin"
```

### Connecting to Amazon OpenSearch Service

```bash
opensearch-benchmark run \
  --workload=geonames \
  --target-hosts=https://my-domain.us-west-2.es.amazonaws.com:443 \
  --pipeline=benchmark-only \
  --client-options="use_ssl:true,verify_certs:true,amazon_aws_log_in:client_option,aws_access_key_id:AKID,aws_secret_access_key:SECRET,region:us-west-2,service:es"
```

For more authentication options, see [Configuring OpenSearch Benchmark]({{site.url}}{{site.baseurl}}/benchmark/user-guide/install-and-configure/configuring-benchmark/).

## Automatic provisioning with from-distribution

OpenSearch Benchmark can download and start a local OpenSearch node automatically:

```bash
opensearch-benchmark run \
  --pipeline=from-distribution \
  --distribution-version=2.19.1 \
  --workload=geonames
```

This downloads the OpenSearch tarball, extracts it, starts a single-node cluster, runs the benchmark, and shuts it down. The distribution is cached at `~/.benchmark/benchmarks/distributions/` so subsequent runs with the same version skip the download.

### JDK requirements

The `from-distribution` pipeline requires a JDK. OpenSearch Benchmark searches for JDK installations using environment variables in this order: `JAVA21_HOME`, `JAVA17_HOME`, `JAVA16_HOME`, `JAVA15_HOME`, ..., `JAVA_HOME`.

If you have JDK 17 installed but OpenSearch Benchmark can't find it:

```bash
export JAVA17_HOME=/path/to/jdk-17
opensearch-benchmark run --pipeline=from-distribution --distribution-version=2.19.1 --workload=geonames
```

### Cluster configuration

The provisioned cluster uses default settings. To customize the cluster, use `--cluster-config` (cluster configuration profiles):

```bash
opensearch-benchmark run \
  --pipeline=from-distribution \
  --distribution-version=2.19.1 \
  --workload=geonames \
  --cluster-config="defaults,4gheap"
```

Available cluster configuration profiles are listed in the cluster configurations repository. Common profiles include:
- `defaults`: Default OpenSearch settings
- `4gheap`: Sets JVM heap to 4 GB
- `16gheap`: Sets JVM heap to 16 GB

## Building from source

For development benchmarking, build OpenSearch from source and benchmark the result:

```bash
opensearch-benchmark run \
  --pipeline=from-sources \
  --revision=latest \
  --workload=geonames
```

Configure the source repository in `benchmark.ini`:

```ini
[source]
remote.repo.url = https://github.com/opensearch-project/OpenSearch.git
opensearch.src.subdir = opensearch
```

### Benchmarking a specific commit

```bash
opensearch-benchmark run \
  --pipeline=from-sources \
  --revision=abc1234 \
  --workload=geonames
```

### Benchmarking with plugins

To include plugins when building from source, use `--opensearch-plugins`:

```bash
opensearch-benchmark run \
  --pipeline=from-sources \
  --revision=latest \
  --workload=geonames \
  --opensearch-plugins="analysis-icu,analysis-phonetic"
```

## Using Docker

```bash
opensearch-benchmark run \
  --pipeline=docker \
  --distribution-version=2.19.1 \
  --workload=geonames
```

Or run OpenSearch Benchmark itself in Docker against an external cluster:

```bash
docker run opensearchproject/opensearch-benchmark run \
  --pipeline=benchmark-only \
  --workload=geonames \
  --target-hosts=host.docker.internal:9200
```

## Multi-node clusters

OpenSearch Benchmark's automatic provisioning (`from-distribution`, `from-sources`, `docker`) creates single-node clusters. For multi-node benchmarks, provision the cluster separately and use `benchmark-only`.

### Using opensearch-cluster-cdk

The [opensearch-cluster-cdk](https://github.com/opensearch-project/opensearch-cluster-cdk) project provisions production-like OpenSearch clusters on AWS:

```bash
npx cdk deploy "*" \
  -c distributionUrl=https://artifacts.opensearch.org/releases/bundle/opensearch/2.19.1/opensearch-2.19.1-linux-x64.tar.gz \
  -c dataNodeCount=3 \
  -c dataInstanceType=r6g.xlarge \
  -c singleNodeCluster=false
```

Then benchmark against the provisioned cluster:

```bash
opensearch-benchmark run \
  --pipeline=benchmark-only \
  --workload=geonames \
  --target-hosts=<cluster-endpoint>:9200
```

### Using Docker Compose

For local multi-node testing, use Docker Compose with the official OpenSearch Docker images:

```yaml
# docker-compose.yml
version: '3'
services:
  opensearch-node1:
    image: opensearchproject/opensearch:2.19.1
    environment:
      - discovery.seed_hosts=opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - DISABLE_SECURITY_PLUGIN=true
    ports:
      - 9200:9200
  opensearch-node2:
    image: opensearchproject/opensearch:2.19.1
    environment:
      - discovery.seed_hosts=opensearch-node1
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - DISABLE_SECURITY_PLUGIN=true
```

```bash
docker compose up -d
opensearch-benchmark run --workload=geonames --target-hosts=localhost:9200
```

## Choosing the right approach

| Approach | Nodes | Setup time | Best for |
|----------|-------|-----------|----------|
| `benchmark-only` + existing cluster | Any | None | Production benchmarks, managed services |
| `from-distribution` | 1 | ~1 min | Quick version comparisons |
| `from-sources` | 1 | ~10 min | Development, pre-release testing |
| `docker` | 1 | ~1 min | Isolated environments |
| CDK | Any | ~15 min | Production-like AWS deployments |
| Docker Compose | 2+ | ~2 min | Local multi-node testing |

For more information about pipelines, see [Pipelines]({{site.url}}{{site.baseurl}}/benchmark/reference/pipelines/).
