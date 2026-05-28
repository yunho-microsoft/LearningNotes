# ElasticSearch

## Basics

### Indexes
An index is the fundamental unit of storage in Elasticsearch.
To store a document, you add it to a specific index. To search, you target one or more indices. Elasticsearch searches all data within them and returns any matching documents.

- In ElasticSearch, index = table, document = row, field = column from a normal SQL database

- ES also has concept of "data stream" in parallel with "Index".

Index or data stream: Use a regular index when you need frequent updates or deletes. For append-only, time series data such as logs, events, and metrics, use a data stream instead, since data streams manage rolling indices automatically.

Summary
https://www.elastic.co/docs/manage-data/data-store

### Documents

Document in ES is a JSON object, just like a row in traditional SQL DB. Each field in Document must have a mapping.

When a document is created in a index(table), all of its fields are indexed based on their mappings.

Frequently used mappings are:
    - Text
    - Keyword


## How search works
Ref: https://www.elastic.co/docs/solutions/search/full-text/how-full-text-works


## When it should be used
Ref: https://www.elastic.co/docs/deploy-manage/production-guidance/general-recommendations

- Return top N result rather than return all result matching a criteria.
- Paging for all results can be compute consuming?

## Pros and cons
### Pros:
- Powerful full text search: it supports complex search
    - Searching on multiple fields

### Cons:
- consume more memory sinch each property is indexed by default