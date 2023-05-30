---
layout: default
title: Search pipelines
nav_order: 100
has_children: true
has_toc: false
---

# Search pipelines

Search pipelines is an experimental feature. For updates on the progress of search pipelines, or if you want to leave feedback that could help improve the feature, see the associated [GitHub issue](https://github.com/opensearch-project/OpenSearch/issues/6278).    
{: .warning}

To integrate result rerankers, query rewriters, and other components that operate on queries or results, you can use _search pipelines_. Search pipelines make it easier for you to process search queries and search results right in OpenSearch, as opposed to building plugins or creating additional logic elsewhere in your application. Thus, you can customize search results without adding complexity to your application. Search pipelines also provide a mechanism for modularizing your processors so you can add and reorder them easily. 

## Terminology

The following is a list of search pipeline terminology:

* _Search request processor_: a component that takes a search request (the query and the metadata passed in the request), performs some operation with or on the search request, and returns a search request.
* _Search response processor_: a component that takes a search response and search request (the query, results, and metadata passed in the request), performs some operation with or on the search response, and returns a search response.
* _Processor_: either a search request processor or a search response processor.
* _Search pipeline_: an ordered list of processors that is integrated into OpenSearch. The pipeline intercepts a query, performs processing on the query, sends it to OpenSearch, intercepts the results, performs processing on the results, and returns them to the calling application, as shown in the following diagram. 

![Search processor diagram]({{site.url}}{{site.baseurl}}/images/search-pipelines.png)

Both request and response processing for the pipeline are performed on the coordinator node so there is no shard-level processing.
{: .note}

## Search request processors

OpenSearch supports the following search request processors:

- [`script`]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/script-processor/): Adds a script that is run on newly indexed documents.
- [`filter_query`]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/filter-query-processor/): Adds a filtering query that is used to filter requests.

## Search response processors

OpenSearch supports the following search response processors:

- [`rename_field`]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/rename-field-processor/): Renames an existing field.

## Creating a search pipeline

Search pipelines are stored in the cluster state. To create a search pipeline, you must configure an ordered list of processors in your OpenSearch cluster. You can have more than one processor of the same type in the pipeline. Each processor has a `tag` identifier that distinguishes it from the others. 

#### Example request

The following request creates a search pipeline with a `filter_query` request processor that uses a term query to return only public messages:

```json
PUT /_search/pipeline/my_pipeline 
{
  "request_processors": [
    {
      "filter_query" : {
        "tag" : "tag1",
        "description" : "This processor is going to restrict to publicly visible documents",
        "query" : {
          "term": {
            "visibility": "public"
          }
        }
      }
    }
  ]
}
```
{% include copy-curl.html %}

You can use the Nodes Search Pipelines API to view the processor types:

```json
GET /_nodes/search_pipelines
```
{% include copy-curl.html %}

The response contains the `search_pipelines` object that lists the possible request and response processors:

<details open markdown="block">
  <summary>
    Response
  </summary>
  {: .text-delta}

```json
{
  "_nodes" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "cluster_name" : "runTask",
  "nodes" : {
    "36FHvCwHT6Srbm2ZniEPhA" : {
      "name" : "runTask-0",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1",
      "version" : "3.0.0",
      "build_type" : "tar",
      "build_hash" : "unknown",
      "roles" : [
        "cluster_manager",
        "data",
        "ingest",
        "remote_cluster_client"
      ],
      "attributes" : {
        "testattr" : "test",
        "shard_indexing_pressure_enabled" : "true"
      },
      "search_pipelines" : {
        "request_processors" : [
          {
            "type" : "filter_query"
          },
          {
            "type" : "script"
          }
        ],
        "response_processors" : [
          {
            "type" : "rename_field"
          }
        ]
      }
    }
  }
}
```
</details>

## Retrieving search pipelines

To retrieve existing search pipeline details, use the Search Pipeline API:

```json
GET /_search/pipeline
```
{% include copy-curl.html %}

The response contains the pipeline that you set up in the previous section:
<details open markdown="block">
  <summary>
    Response
  </summary>
  {: .text-delta}

```json
{
  "my_pipeline" : {
    "request_processors" : [
      {
        "filter_query" : {
          "tag" : "tag1",
          "description" : "This processor is going to restrict to publicly visible documents",
          "query" : {
            "term" : {
              "visibility" : "public"
            }
          }
        }
      }
    ]
  }
}
```
</details>

## Using a search pipeline

To search with a pipeline, specify the pipeline name in the `search_pipeline` query parameter:

```json
GET /my_index/_search?search_pipeline=my_pipeline
```
{% include copy-curl.html %}

For a complete example of using a search pipeline with a `filter_query` processor, see [`filter_query` processor example]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/filter-query-processor#example).

## Default search pipeline

For convenience, you can set a default search pipeline on an index. Once your index has a default pipeline, you don't need to specify the `search_pipeline` query parameter with every search request.

### Setting a default search pipeline on an index

To set a default search pipeline on an index, specify the `index.search.default_pipeline` in the index's settings:

```json
PUT /my_index/_settings 
{
  "index.search.default_pipeline" : "my_pipeline"
}
```
{% include copy-curl.html %}

After setting the default pipeline on `my_index`, you can try the same search for all documents:

```json
GET /my_index/_search
```
{% include copy-curl.html %}

The response contains only the public document, indicating that the pipeline was applied by default:

<details open markdown="block">
  <summary>
    Response
  </summary>
  {: .text-delta}

```json
{
  "took" : 19,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.0,
    "hits" : [
      {
        "_index" : "my_index",
        "_id" : "1",
        "_score" : 0.0,
        "_source" : {
          "message" : "This is a public message",
          "visibility" : "public"
        }
      }
    ]
  }
}
```
</details>

### Disabling the default pipeline for a request

If you want to run a search request without applying the default pipeline, you can set the `search_pipeline` query parameter to `_none`:

```json
GET /my_index/_search?search_pipeline=_none
```
{% include copy-curl.html %}

### Removing the default pipeline

To remove the default pipeline from an index, set it to `null` or `_none`:

```json
PUT /my_index/_settings 
{
  "index.search.default_pipeline" : null
}
```
{% include copy-curl.html %}

```json
PUT /my_index/_settings 
{
  "index.search.default_pipeline" : "_none"
}
```
{% include copy-curl.html %}

## Updating a search pipeline

You can update a search pipeline dynamically using the Search Pipeline API. To update an existing search processor, make sure to specify its ID in the `tag` field.

#### Example request

The following request adds a `rename_field` response processor to `my_pipeline`:

```json
PUT /_search/pipeline/my_pipeline
{
  "request_processors": [
    {
      "filter_query": {
        "tag": "tag1",
        "description": "This processor returns only publicly visible documents",
        "query": {
          "term": {
            "visibility": "public"
          }
        }
      }
    }
  ],
  "response_processors": [
    {
      "rename_field": {
        "field": "message",
        "target_field": "notification"
      }
    }
  ]
}
```
{% include copy-curl.html %}

## Search pipeline versions

You can specify a version for your pipeline when creating or updating it. 