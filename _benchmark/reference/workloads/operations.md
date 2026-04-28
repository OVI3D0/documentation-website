---
layout: default
title: Operations
parent: Workload reference
grand_parent: Reference
nav_order: 100
redirect_from:
  - /benchmark/user-guide/understanding-workloads/common-operations/
---

# Operations

[Test procedures]({{site.url}}{{site.baseurl}}/benchmark/user-guide/understanding-workloads/anatomy-of-a-workload#_operations-and-_test-procedures) use a variety of operations, found inside the `operations` directory of a workload. The `operations` element contains a list of all available operations for specifying a schedule.

<!-- vale off -->
## bulk
<!-- vale on -->

The `bulk` operation type allows you to run [bulk]({{site.url}}{{site.baseurl}}/api-reference/document-apis/bulk/) requests as a task.

### Usage

The following example shows a `bulk` operation type with a `bulk-size` of `5000` documents:

```yml
{
  "name": "index-append",
  "operation-type": "bulk",
  "bulk-size": 5000
}
```

### Split documents among clients

When you have multiple `clients`, OpenSearch Benchmark splits each document based on the set number of clients. Having multiple `clients` parallelizes the bulk index operations but doesn't preserve the ingestion order of each document. For example, if `clients` is set to `2`, one client indexes the document starting from the beginning, while the other client indexes the document starting from the middle.

If there are multiple documents or corpora, OpenSearch Benchmark attempts to index all documents in parallel in two ways:

1. Each client starts at a different point in the corpus. For example, in a workload with 2 corpora and 5 clients, clients 1, 3, and 5 begin with the first corpus, whereas clients 2 and 4 start with the second corpus.
2. Each client is assigned to multiple documents. Client 1 starts with the first split of the first document of the first corpus. Then it moves to the first split of the first document of the second corpus, and so on.

### Configuration options

Use the following options to customize the `bulk` operation.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`bulk-size` | Yes | Number | Specifies the number of documents to be ingested in the bulk request.
`ingest-percentage` | No | Range [0, 100] | Defines the portion of the document corpus to be indexed. Valid values are numbers between 0 and 100.
`corpora` | No | List | Defines which document corpus names should be targeted by the bulk operation. Only needed if the `corpora` section contains more than one document corpus and you don’t want to index all of them during the bulk request.
`indices` | No | List | Defines which indexes should be used in the bulk index operation. OpenSearch Benchmark only selects document files that have a matching `target-index`.
`batch-size` | No | Number | Defines how many documents OpenSearch Benchmark reads simultaneously. This is an expert setting and is only meant to avoid accidental bottlenecks for very small bulk sizes. If you want to benchmark with a `bulk-size` of `1`, you should set a higher `batch-size`.
`pipeline` | No | String | Defines which existing ingest pipeline to use.
`conflicts` | No | String | Defines the type of index `conflicts` to simulate. If not specified, none are simulated. Valid values are ‘sequential’, which replaces a document ID with a sequentially increasing document ID, and ‘random’, which replaces a document ID with a random document ID.
`conflict-probability` | No | Percentage | Defines how many of the documents are replaced when a conflict exists. Combining `conflicts=sequential` and `conflict-probability=0` makes OpenSearch Benchmark generate the index ID itself instead of using OpenSearch's automatic ID generation. Valid values are numbers between 0 and 100. Default is `25%`.
`on-conflict` | No | String |  Determines whether OpenSearch should use the action `index` or `update` index for ID conflicts. Default is `index`, which creates a new index during ID conflicts.
`recency` | No | Number | Uses a number between 0 and 1 to indicate recency. A recency closer to `1` biases conflicting IDs toward more recent IDs. A recency closer to 0 considers all IDs for ID conflicts.
`detailed-results` | No | Boolean | Records more detailed [metadata](#metadata) for bulk requests. As OpenSearch Benchmark analyzes the corresponding bulk response in more detail, additional overhead may be incurred, which can skew measurement results. This property must be set to `true` so that OpenSearch Benchmark logs individual bulk request failures.
`timeout` | No | Duration | Defines the amount of time (in minutes) that OpenSearch waits per action until completing the processing of the following operations: automatic index creation, dynamic mapping updates, and waiting for active shards. Default is `1m`.
`refresh` | No | String | Controls OpenSearch refresh behavior for bulk requests that use the `refresh` Bulk API query parameter. Valid values are `true`, which refreshes target shards in the background; `wait_for`, which blocks bulk requests until affected shards have been refreshed; and `false`, which uses the default refresh behavior.

### Metadata

The `bulk` operation always returns the following metadata:

- `index`: The name of the affected index. If an index cannot be derived, it returns `null`.
- `weight`: An operation-agnostic representation of the bulk size, denoted by `units`.
- `unit`: The unit used to interpret `weight`.
- `success`: A Boolean indicating whether the `bulk` request succeeded.
- `success-count`: The number of successfully processed bulk items for the request. This value is determined when there are errors or when the `bulk-size` has been specified in the documents.
- `error-count`: The number of failed bulk items for the request.
- `took`: The value of the `took` property in the bulk response.

If `detailed-results` is `true`, the following metadata is returned:

- `ops`: A nested document with the operation name as its key, such as `index`, `update`, or `delete`, and various counts as values. `item-count` contains the total number of items for this key. Additionally, OpenSearch Benchmark returns a separate counter for each result, for example, a result for the number of created items or the number of deleted items.
- `shards_histogram`: An array of hashes, each of which has two keys. The `item-count` key contains the number of items to which a shard distribution applies. The `shards` key contains a hash with the actual distribution of `total`, `successful`, and `failed` shards.
- `bulk-request-size-bytes`: The total size of the bulk request body, in bytes.
- `total-document-size-bytes`: The total size of all documents within the bulk request body, in bytes.

<!-- vale off -->
## create-index
<!-- vale on -->

The `create-index` operation runs the [Create Index API]({{site.url}}{{site.baseurl}}/api-reference/index-apis/create-index/). It supports the following two index creation modes:

- Creating all indexes specified in the workloads `indices` section
- Creating one specific index defined within the operation itself

### Usage

The following example creates all indexes defined in the `indices` section of the workload. It uses all of the index settings defined in the workload but overrides the number of shards:

```yml
{
  "name": "create-all-indices",
  "operation-type": "create-index",
  "settings": {
    "index.number_of_shards": 1
  },
  "request-params": {
    "wait_for_active_shards": "true"
  }
}
```

The following example creates a new index with all index settings specified in the operation body:

```yml
{
  "name": "create-an-index",
  "operation-type": "create-index",
  "index": "people",
  "body": {
    "settings": {
      "index.number_of_shards": 0
    },
    "mappings": {
      "docs": {
        "properties": {
          "name": {
            "type": "text"
          }
        }
      }
    }
  }
}
```

### Configuration options

Use the following options when creating all indexes from the `indices` section of a workload.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`settings` | No | Array |  Specifies additional index settings to be merged with the index settings specified in the `indices` section of the workload.
`request-params` | No | List of settings | Contains any request parameters allowed by the Create Index API. OpenSearch Benchmark does not attempt to serialize the parameters and passes them in their current state.

Use the following options when creating a single index in the operation.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | Yes | String | The index name.
`body` | No | Request body | The request body for the Create Index API. For more information, see [Create Index API](/api-reference/index-apis/create-index/).
`request-params` | No | List of settings | Contains any request parameters allowed by the Create Index API. OpenSearch Benchmark does not attempt to serialize the parameters and passes them in their current state.

### Metadata

The `create-index` operation returns the following metadata:

`weight`: The number of indexes created by the operation.
`unit`: Always `ops`, indicating the number of operations inside the workload.
`success`: A Boolean indicating whether the operation has succeeded.

<!-- vale off -->
## delete-index
<!-- vale on -->

The `delete-index` operation runs the [Delete Index API]({{site.url}}{{site.baseurl}}/api-reference/index-apis/delete-index/). As with the [`create-index`](#create-index) operation, you can delete all indexes found in the `indices` section of the workload or delete one or more indexes based on the string passed in the `index` setting.

### Usage

The following example deletes all indexes found in the `indices` section of the workload:

```yml
{
  "name": "delete-all-indices",
  "operation-type": "delete-index"
}
```

The following example deletes all `logs_*` indexes:

```yml
{
  "name": "delete-logs",
  "operation-type": "delete-index",
  "index": "logs-*",
  "only-if-exists": false,
  "request-params": {
    "expand_wildcards": "all",
    "allow_no_indices": "true",
    "ignore_unavailable": "true"
  }
}
```

### Configuration options

Use the following options when deleting all indexes indicated in the `indices` section of the workload.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`only-if-exists` | No | Boolean | Decides whether an existing index should be deleted. Default is `true`.
`request-params` | No | List of settings | Contains any request parameters allowed by the Create Index API. OpenSearch Benchmark does not attempt to serialize the parameters and passes them in their current state.

Use the following options if you want to delete one or more indexes based on the pattern indicated in the `index` option.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | Yes | String | The index or indexes that you want to delete.
`only-if-exists` | No | Boolean | Decides whether an index should be deleted when the index exists. Default is `true`.
`request-params` | No | List of settings | Contains any request parameters allowed by the Create Index API. OpenSearch Benchmark does not attempt to serialize the parameters and passes them in their current state.

### Metadata

The `delete-index` operation returns the following metadata:

`weight`: The number of indexes created by the operation.
`unit`: Always `ops`, for the number of operations inside the workload.
`success`: A Boolean indicating whether the operation has succeeded.

<!-- vale off -->
## cluster-health
<!-- vale on -->

The `cluster-health` operation runs the [Cluster Health API]({{site.url}}{{site.baseurl}}/api-reference/cluster-api/cluster-health/), which checks the cluster health status and returns the expected status according to the parameters set for `request-params`. If an unexpected cluster health status is returned, then the operation reports a failure. You can use the `--on-error` option in the OpenSearch Benchmark `run` command to control how OpenSearch Benchmark behaves when the health check fails.


### Usage

The following example creates a `cluster-health` operation that checks for a `green` health status on any `log-*` indexes:

```yml
{
  "name": "check-cluster-green",
  "operation-type": "cluster-health",
  "index": "logs-*",
  "request-params": {
    "wait_for_status": "green",
    "wait_for_no_relocating_shards": "true"
  },
  "retry-until-success": true
}

```

### Configuration options

Use the following options with the `cluster-health` operation.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | Yes | String | The index or indexes you want to assess.
`request-params` | No | List of settings | Contains any request parameters allowed by the Cluster Health API. OpenSearch Benchmark does not attempt to serialize the parameters and passes them in their current state.

### Metadata

The `cluster-health` operation returns the following metadata:

`weight`: The number of indexes the `cluster-health` operation assesses. Alwasys `1`, since the operation runs once per index.
`unit`: Always `ops`, for the number of operations inside the workload.
`success`: A Boolean indicating whether the operation has succeeded.
- `cluster-status`: The current cluster status.
- `relocating-shards`: The number of shards currently relocating to a different node.

<!-- vale off -->
## refresh
<!-- vale on -->

The `refresh` operation runs the Refresh API. The `operation` returns no metadata.

### Usage

The following example refreshes all `logs-*` indexes:

```yml
{
 "name": "refresh",
 "operation-type": "refresh",
 "index": "logs-*"
}
```

### Configuration options

The `refresh` operation uses the following options.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The names of the indexes or data streams to refresh.

<!-- vale off -->
## search
<!-- vale on -->

The `search` operation runs the [Search API]({{site.url}}{{site.baseurl}}/api-reference/search/), which you can use to run queries in OpenSearch Benchmark indexes.

### Usage

The following example runs a `match_all` query inside the `search` operation:

```yml
{
  "name": "default",
  "operation-type": "search",
  "body": {
    "query": {
      "match_all": {}
    }
  },
  "request-params": {
    "_source_include": "some_field",
    "analyze_wildcard": "false"
  }
}
```

### Configuration options

The `search` operation uses the following options.

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The indexes or data streams targeted by the query. This option is needed only when the `indices` section contains two or more indexes. Otherwise, OpenSearch Benchmark automatically derives the index or data stream to use. Specify `"index": "_all"` to query against all indexes in the workload.
`cache` | No | Boolean | Specifies whether to use the query request cache. OpenSearch Benchmark defines no value. The default depends on the benchmark candidate settings and the OpenSearch version.
`request-params` | No | List of settings | Contains any request parameters allowed by the Search API.
`body` | Yes | Request body | Indicates which query and query parameters to use.
`detailed-results` | No | Boolean | Records more detailed metadata about queries. When set to `true`, additional overhead may be incurred, which can skew measurement results. This option does not work with `scroll` queries.
`results-per-page` | No | Integer | Specifies the number of documents to retrieve per page. This maps to the Search API `size` parameter and can be used for scroll and non-scroll searches. Default is `10`.

### Metadata

The following metadata is always returned:

- `weight`: The “weight” of an operation. Always `1` for regular queries and the number of retrieved pages for scroll queries.
- `unit`: The unit used to interpret weight, which is `ops` for regular queries and `pages` for scroll queries.
- `success`: A Boolean indicating whether the query has succeeded.

If `detailed-results` is set to `true`, the following metadata is also returned:

- `hits`: The total number of hits for the query.
- `hits_relation`: Whether the number of hits is accurate (eq) or a lower bound of the actual hit count (gte).
- `timed_out`: Whether the query has timed out. For scroll queries, this flag is `true` if the flag was `true` for any of the queries issued.
 - `took`: The value of the `took` property in the query response. For scroll queries, the value is the sum of all `took` values in all query responses.



<!-- vale off -->
## force-merge
<!-- vale on -->

The `force-merge` operation runs the [Force Merge API]({{site.url}}{{site.baseurl}}/api-reference/index-apis/force-merge/).

### Usage

```json
{
  "name": "force-merge",
  "operation-type": "force-merge",
  "index": "_all",
  "request-params": {
    "max_num_segments": 1
  }
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The index or indexes to force merge. Default is `_all`.
`request-params` | No | Object | Request parameters passed to the Force Merge API, such as `max_num_segments`.
`mode` | No | String | Set to `polling` to wait asynchronously for the merge to complete.

This is an administrative operation. Metrics are not reported by default. Reporting can be forced by setting `include-in-reporting` to `true`.

<!-- vale off -->
## index-stats
<!-- vale on -->

The `index-stats` operation runs the [Index Stats API]({{site.url}}{{site.baseurl}}/api-reference/index-apis/stats/).

### Usage

```json
{
  "name": "index-stats",
  "operation-type": "index-stats",
  "index": "_all",
  "condition": {
    "path": "_all.total.merges.current",
    "expected-value": 0
  },
  "retry-until-success": true
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The index or indexes to retrieve statistics for. Default is `_all`.
`condition` | No | Object | A condition to check in the response. Contains `path` (dot-notation path in the response) and `expected-value`.
`request-params` | No | Object | Request parameters passed to the Stats API.

<!-- vale off -->
## node-stats
<!-- vale on -->

The `node-stats` operation runs the [Nodes Stats API]({{site.url}}{{site.baseurl}}/api-reference/nodes-apis/nodes-stats/).

### Usage

```json
{
  "name": "node-stats",
  "operation-type": "node-stats"
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`request-params` | No | Object | Request parameters passed to the Nodes Stats API.

<!-- vale off -->
## vector-search
<!-- vale on -->

The `vector-search` operation runs k-NN vector search queries and optionally computes recall metrics by comparing results against ground truth neighbors.

### Usage

```json
{
  "name": "knn-search",
  "operation-type": "vector-search",
  "index": "vectors",
  "k": 100,
  "body": {
    "query": {
      "knn": {
        "embedding": {
          "vector": [0.1, 0.2, 0.3],
          "k": 100
        }
      }
    }
  }
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The target index.
`body` | Yes | Object | The search request body containing the k-NN query.
`k` | No | Integer | The number of nearest neighbors to retrieve. Used for recall calculation.
`detailed-results` | No | Boolean | Records detailed metadata.
`calculate-recall` | No | Boolean | Whether to compute recall@k and recall@1 against ground truth neighbors. Default is `true`.
`neighbors` | No | Array | Ground truth neighbor IDs for recall calculation. Typically provided by the workload's param source.

### Metadata

- `weight`: Always `1`.
- `unit`: Always `ops`.
- `success`: Whether the search succeeded.
- `recall@k`: Recall at k (if ground truth neighbors are provided).
- `recall@1`: Recall at 1 (if ground truth neighbors are provided).

<!-- vale off -->
## bulk-vector-data-set
<!-- vale on -->

The `bulk-vector-data-set` operation bulk-indexes vector data from HDF5 or BigANN dataset files. Supports individual document retry on partial failures.

### Usage

```json
{
  "name": "bulk-vectors",
  "operation-type": "bulk-vector-data-set",
  "bulk-size": 500,
  "index": "vectors"
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`bulk-size` | No | Integer | Number of documents per bulk request.
`index` | No | String | The target index.
`retries` | No | Integer | Number of retry attempts on failure. Default is `3`.
`retry-wait-period` | No | Number | Initial wait period between retries in seconds. Default is `0.5`.
`retry-max-wait-period` | No | Number | Maximum wait period between retries in seconds (exponential backoff capped at this value). Default is `60`.
`detailed-results` | No | Boolean | Records detailed per-document success/failure metadata.

<!-- vale off -->
## put-pipeline
<!-- vale on -->

The `put-pipeline` operation creates or updates an [ingest pipeline]({{site.url}}{{site.baseurl}}/ingest-pipelines/).

### Usage

```json
{
  "name": "define-pipeline",
  "operation-type": "put-pipeline",
  "id": "my-pipeline",
  "body": {
    "description": "My ingest pipeline",
    "processors": [
      {
        "set": {
          "field": "ingest_time",
          "value": "{{_ingest.timestamp}}"
        }
      }
    ]
  }
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`id` | Yes | String | The pipeline ID.
`body` | Yes | Object | The pipeline definition.

This is an administrative operation. Metrics are not reported by default.

<!-- vale off -->
## delete-pipeline
<!-- vale on -->

The `delete-pipeline` operation deletes an ingest pipeline.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`id` | Yes | String | The pipeline ID to delete.

This is an administrative operation. Metrics are not reported by default.

<!-- vale off -->
## create-search-pipeline
<!-- vale on -->

The `create-search-pipeline` operation creates a [search pipeline]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`id` | Yes | String | The search pipeline ID.
`body` | Yes | Object | The search pipeline definition.

This is an administrative operation. Metrics are not reported by default.

<!-- vale off -->
## put-settings
<!-- vale on -->

The `put-settings` operation updates [cluster settings]({{site.url}}{{site.baseurl}}/api-reference/cluster-api/cluster-settings/).

### Usage

```json
{
  "name": "increase-watermarks",
  "operation-type": "put-settings",
  "body": {
    "transient": {
      "cluster.routing.allocation.disk.watermark.low": "95%"
    }
  }
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`body` | Yes | Object | The cluster settings to apply.

This is an administrative operation. Metrics are not reported by default.

<!-- vale off -->
## create-data-stream
<!-- vale on -->

The `create-data-stream` operation creates a [data stream]({{site.url}}{{site.baseurl}}/opensearch/data-streams/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`data-stream` | Yes | String | The data stream name.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## delete-data-stream
<!-- vale on -->

The `delete-data-stream` operation deletes a data stream.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`data-stream` | Yes | String | The data stream name.
`only-if-exists` | No | Boolean | Only delete if the data stream exists. Default is `true`.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## create-composable-template
<!-- vale on -->

The `create-composable-template` operation creates a [composable index template]({{site.url}}{{site.baseurl}}/im-plugin/index-templates/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`template` | Yes | String | The template name.
`body` | Yes | Object | The template definition.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## delete-composable-template
<!-- vale on -->

The `delete-composable-template` operation deletes a composable index template.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`template` | Yes | String | The template name.
`only-if-exists` | No | Boolean | Only delete if the template exists. Default is `true`.
`delete-matching-indices` | No | Boolean | Also delete indices matching the template pattern. Default is `false`.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## create-component-template
<!-- vale on -->

The `create-component-template` operation creates a [component template]({{site.url}}{{site.baseurl}}/im-plugin/index-templates/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`template` | Yes | String | The component template name.
`body` | Yes | Object | The component template definition.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## delete-component-template
<!-- vale on -->

The `delete-component-template` operation deletes a component template.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`template` | Yes | String | The component template name.
`only-if-exists` | No | Boolean | Only delete if the template exists. Default is `true`.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## create-index-template
<!-- vale on -->

The `create-index-template` operation creates a legacy index template.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`template` | Yes | String | The template name.
`body` | Yes | Object | The template definition.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## delete-index-template
<!-- vale on -->

The `delete-index-template` operation deletes a legacy index template.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`template` | Yes | String | The template name.
`only-if-exists` | No | Boolean | Only delete if the template exists. Default is `true`.
`delete-matching-indices` | No | Boolean | Also delete indices matching the template pattern. Default is `false`.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## shrink-index
<!-- vale on -->

The `shrink-index` operation uses the [Shrink Index API]({{site.url}}{{site.baseurl}}/api-reference/index-apis/shrink-index/) to reduce the number of primary shards in an index.

### Usage

```json
{
  "name": "shrink-index",
  "operation-type": "shrink-index",
  "source-index": "my-index",
  "target-index": "my-index-shrunk",
  "target-body": {
    "settings": {
      "index.number_of_shards": 1,
      "index.codec": "best_compression"
    }
  }
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`source-index` | Yes | String | The source index to shrink.
`target-index` | Yes | String | The name for the shrunk index.
`target-body` | No | Object | Settings and mappings for the target index.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## create-snapshot-repository
<!-- vale on -->

The `create-snapshot-repository` operation creates a [snapshot repository]({{site.url}}{{site.baseurl}}/opensearch/snapshots/snapshot-restore/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`repository` | Yes | String | The repository name.
`body` | Yes | Object | The repository definition (type, settings).
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## delete-snapshot-repository
<!-- vale on -->

The `delete-snapshot-repository` operation deletes a snapshot repository.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`repository` | Yes | String | The repository name to delete.

<!-- vale off -->
## create-snapshot
<!-- vale on -->

The `create-snapshot` operation creates a [snapshot]({{site.url}}{{site.baseurl}}/opensearch/snapshots/snapshot-restore/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`repository` | Yes | String | The repository name.
`snapshot` | Yes | String | The snapshot name.
`body` | No | Object | The snapshot definition.
`wait-for-completion` | No | Boolean | Whether to wait for the snapshot to complete. Default is `true`.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## wait-for-snapshot-create
<!-- vale on -->

The `wait-for-snapshot-create` operation polls the snapshot status until it completes.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`repository` | Yes | String | The repository name.
`snapshot` | Yes | String | The snapshot name.

<!-- vale off -->
## restore-snapshot
<!-- vale on -->

The `restore-snapshot` operation restores a snapshot.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`repository` | Yes | String | The repository name.
`snapshot` | Yes | String | The snapshot name.
`body` | No | Object | The restore request body.
`request-params` | No | Object | Additional request parameters.

<!-- vale off -->
## wait-for-recovery
<!-- vale on -->

The `wait-for-recovery` operation waits until index recovery completes by polling the [Index Recovery API]({{site.url}}{{site.baseurl}}/api-reference/index-apis/recover/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The index to monitor recovery for. Default is `_all`.
`request-params` | No | Object | Additional request parameters.
`retry-until-success` | No | Boolean | Whether to keep retrying until recovery completes. Default is `false`.

<!-- vale off -->
## submit-async-search
<!-- vale on -->

The `submit-async-search` operation submits an [asynchronous search]({{site.url}}{{site.baseurl}}/search-plugins/async/) request.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | No | String | The target index.
`body` | Yes | Object | The search request body.
`request-params` | No | Object | Additional request parameters.
`cache` | No | Boolean | Whether to cache results.

<!-- vale off -->
## get-async-search
<!-- vale on -->

The `get-async-search` operation retrieves the results of a previously submitted async search.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`retrieve-results-for` | Yes | String | The name of the `submit-async-search` operation to retrieve results for.

<!-- vale off -->
## delete-async-search
<!-- vale on -->

The `delete-async-search` operation deletes an async search result.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`retrieve-results-for` | Yes | String | The name of the `submit-async-search` operation whose results to delete.

<!-- vale off -->
## create-point-in-time
<!-- vale on -->

The `create-point-in-time` operation creates a [Point in Time (PIT)]({{site.url}}{{site.baseurl}}/search-plugins/point-in-time/) for an index.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`index` | Yes | String | The target index.
`keep-alive` | No | String | How long to keep the PIT alive (e.g., `5m`).

<!-- vale off -->
## delete-point-in-time
<!-- vale on -->

The `delete-point-in-time` operation deletes a PIT.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`pit-id` | No | String | The PIT ID to delete. If not specified, uses the ID from the most recent `create-point-in-time` operation.

<!-- vale off -->
## list-all-point-in-time
<!-- vale on -->

The `list-all-point-in-time` operation lists all active PITs on the cluster.

<!-- vale off -->
## create-transform
<!-- vale on -->

The `create-transform` operation creates an [index transform]({{site.url}}{{site.baseurl}}/im-plugin/index-transforms/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`transform-id` | Yes | String | The transform ID.
`body` | Yes | Object | The transform definition.

<!-- vale off -->
## start-transform
<!-- vale on -->

The `start-transform` operation starts a previously created transform.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`transform-id` | Yes | String | The transform ID to start.

<!-- vale off -->
## wait-for-transform
<!-- vale on -->

The `wait-for-transform` operation polls the transform status until it completes.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`transform-id` | Yes | String | The transform ID to monitor.
`timeout` | No | Number | Maximum time in seconds to wait for completion. Default is `300`.

<!-- vale off -->
## train-knn-model
<!-- vale on -->

The `train-knn-model` operation trains a k-NN model using the [Train Model API]({{site.url}}{{site.baseurl}}/vector-search/api/knn/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`body` | Yes | Object | The training request body including training index, field, and model parameters.
`timeout` | No | Number | Maximum time in seconds to wait for training to complete. Default is `300`.

<!-- vale off -->
## delete-knn-model
<!-- vale on -->

The `delete-knn-model` operation deletes a trained k-NN model.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`model-id` | Yes | String | The model ID to delete.

<!-- vale off -->
## register-ml-model
<!-- vale on -->

The `register-ml-model` operation registers a machine learning model using the [ML Commons API]({{site.url}}{{site.baseurl}}/ml-commons-plugin/api/model-apis/register-model/).

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`body` | Yes | Object | The model registration request body.

<!-- vale off -->
## deploy-ml-model
<!-- vale on -->

The `deploy-ml-model` operation deploys (loads) a registered ML model.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`model-id` | Yes | String | The model ID to deploy. Can reference a model registered in a previous `register-ml-model` operation.
`timeout` | No | Number | Maximum time in seconds to wait for deployment. Default is `300`.

<!-- vale off -->
## create-ml-connector
<!-- vale on -->

The `create-ml-connector` operation creates an [ML connector]({{site.url}}{{site.baseurl}}/ml-commons-plugin/remote-models/connectors/) for remote model integration.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`body` | Yes | Object | The connector definition.

<!-- vale off -->
## delete-ml-connector
<!-- vale on -->

The `delete-ml-connector` operation deletes an ML connector.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`connector-id` | Yes | String | The connector ID to delete.

<!-- vale off -->
## delete-ml-model
<!-- vale on -->

The `delete-ml-model` operation deletes a registered ML model.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`model-id` | Yes | String | The model ID to delete.

<!-- vale off -->
## raw-request
<!-- vale on -->

The `raw-request` operation sends an arbitrary HTTP request to OpenSearch. Use this for operations not covered by a dedicated operation type.

### Usage

```json
{
  "name": "check-shard-count",
  "operation-type": "raw-request",
  "method": "GET",
  "path": "/_cat/shards?v",
  "body": {}
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`method` | No | String | HTTP method (`GET`, `POST`, `PUT`, `DELETE`). Default is `GET`.
`path` | Yes | String | The URL path (relative to the cluster root).
`body` | No | Object | The request body.
`request-params` | No | Object | Query string parameters.
`headers` | No | Object | HTTP headers to include.

<!-- vale off -->
## sleep
<!-- vale on -->

The `sleep` operation pauses execution for a specified duration.

### Usage

```json
{
  "name": "wait-before-search",
  "operation-type": "sleep",
  "duration": 30
}
```

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`duration` | Yes | Number | Sleep duration in seconds.

<!-- vale off -->
## composite
<!-- vale on -->

The `composite` operation runs multiple operations sequentially within a single task. This is useful for operations that must execute as a unit, such as creating a PIT, searching with it, and deleting it.

### Usage

```json
{
  "name": "pit-search-workflow",
  "operation-type": "composite",
  "requests": [
    { "operation-type": "create-point-in-time", "index": "my-index" },
    { "operation-type": "search", "body": { "query": { "match_all": {} } } },
    { "operation-type": "delete-point-in-time" }
  ]
}
```

<!-- vale off -->
## produce-stream-message
<!-- vale on -->

The `produce-stream-message` operation produces messages to a Kafka topic for streaming ingestion benchmarks.

### Configuration options

Parameter | Required | Type | Description
:--- | :--- | :--- | :---
`body` | Yes | Object | The message body to produce.
`topic` | No | String | The Kafka topic name. Derived from workload configuration if not specified.

<!-- vale off -->
## proto-bulk
<!-- vale on -->

The `proto-bulk` operation sends bulk index requests using the gRPC transport instead of HTTP REST. Requires gRPC to be enabled on the target cluster.

### Configuration options

Same as `bulk`, but uses gRPC serialization (Protocol Buffers) instead of JSON over HTTP. Use `--grpc-target-hosts` to specify the gRPC endpoint.

<!-- vale off -->
## proto-search
<!-- vale on -->

The `proto-search` operation sends search requests using gRPC transport.

### Configuration options

Same as `search`, but uses gRPC. Use `--grpc-target-hosts` to specify the gRPC endpoint.

<!-- vale off -->
## proto-vector-search
<!-- vale on -->

The `proto-vector-search` operation sends vector search requests using gRPC transport.

### Configuration options

Same as `vector-search`, but uses gRPC. Use `--grpc-target-hosts` to specify the gRPC endpoint.
