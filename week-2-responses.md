
# Level 1

## Q: How long did it take to index the 1.2M product data set?  What docs/sec indexing rate did you see?
Answer: *I'm using gitpod and in my case it took 9 minutes and 53 Seconds with ingestion rate of 2.2K docs/sec (avg) and 3.52K docs/sec (peak)*

## Q: Looking at the metrics dashboard, what queries/sec rate are you getting?
Answer: *About 62 query/secs in gitpod instance*

## Q: What resource(s) appear to be the constraining factor?
Answer: *CPU is the bottleneck as it is sustained at 100%*


# Level 2

> CPU:1 MEMORY: 8GB

## Q: How long did it take to index the 1.2M product data set?  What docs/sec indexing rate did you see?
Answer: *Using gitpod and and it took ~9 minutes with ingestion rate of 2.35K docs/sec (avg) and 4.1K docs/sec (peak)*

## Q: Looking at the metrics dashboard, what queries/sec rate are you getting?
Answer: *About 65 query/secs in gitpod instance, so not really a meaningful difference as the search is CPU bound mostly*

## Q: What resource(s) appear to be the constraining factor?
Answer: *CPU is the bottleneck as it is sustained at 100%*



> CPU:4 MEMORY: 8GB

## Q: How long did it take to index the 1.2M product data set?  What docs/sec indexing rate did you see?
Answer: *Using gitpod and and it took 8.9 minutes with ingestion rate of 2.38K docs/sec (avg) and 4.11K docs/sec (peak)*

## Q: Looking at the metrics dashboard, what queries/sec rate are you getting?
Answer: *About 201 query/secs in gitpod instance, this shows search is CPU bound mostly*

## Q: What resource(s) appear to be the constraining factor?
Answer: *CPU is the bottleneck as it was closer to 100%*


> CPU:4 MEMORY 4GB

## Q: How long did it take to index the 1.2M product data set?  What docs/sec indexing rate did you see?
Answer: *Using gitpod and and it took 8.46 minutes with ingestion rate of 2.38K docs/sec (avg) and 3.9K docs/sec (peak)*

## Q: Looking at the metrics dashboard, what queries/sec rate are you getting?
Answer: *About 175 query/secs in gitpod instance, it is mostly CPU but extra Memory seems useful as when we decrease the memory throughput decreases a bit, and CPU is not 100% always*

## Q: What resource(s) appear to be the constraining factor?
Answer: *CPU may not be totally the bottleneck, I don't see much GC happening, or cache evictions either so memory may not be the bottleneck either, I speculate disk speed might be the issue as there are many cache misses but I'm not 100% sure*



# Level 3: Indexing + Querying Together

## Q: What is the impact on your query throughput (QPS) and indexing throughput (docs/sec)?
Answer: * I used 4 vCPU and 8GB RAM, and concurrent indexing/searching has adversary effect on performance. Search rate can barely make it to 50 q/sec but indexing was not terribly impacted, it was around 2.3K docs/sec. The CPU became the bottleneck again as now we have 2 threads for Search and ~4 for indexing (sometimes refresh borrow one thread) so we don't get great search speed while indexing is still good. In total it took 9.15 mins to index. Once the indexing finished, search rate climned to 175 q/sec.


# Level 4: Optional

I'm surprised how robust Opensearch is, I deliberately turned on field data caching on a text field `shortDescription` (analyzed) so cardinality would be huge.

```json
PUT bbuy_products/_mapping
{
  "properties": {
    "shortDescription": { 
      "type":     "text",
      "fielddata": true,
      "analyzer": "english"
    }
  }
}
```

And after that I ran the aggregation query with size 10K so that I aggregate count of 10K distinct terms, which should really create a lot of strain on field data cache size, and indeed id created a big cache. 
However query returned the result in less than 0.3 seconds with 117k lines of JSON and aggregation for individual tokens. Even with a single node the amount of things we can do is incredible. I didn't even triggered circuit breaker for filed data cache. 


I've also tried some `regexp` queries with 10K size and highlighting on see below
```json
GET /bbuy_products/_search
{
  "size": 10000, 
  "query": {
    "regexp": {
      "features": {
        "value": "d...y",
        "flags": "ALL",
        "case_insensitive": true,
        "max_determinized_states": 10000,
        "rewrite": "constant_score"
      }
    }
  },
  "highlight": {
    "fields": {
      "features": {}
    }
  }
}
```

this took a bit over 1.2 seconds to return, with 8468 matches, however the size of data was absolutely very big, 1.4 million lines. So I admit it's difficult to trigger circut breaker even it is a single node flimsy server. 

