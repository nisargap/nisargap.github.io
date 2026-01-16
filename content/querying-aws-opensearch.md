+++
title = "Making Sense of OpenSearch Queries"
date = 2026-01-16
description = "AWS OpenSearch has powerful query capabilities, but the learning curve is steep. Here's how to construct queries that actually work."
[taxonomies]
tags = ["aws", "opensearch", "elasticsearch", "search"]
categories = ["Backend"]
+++

OpenSearch query syntax looks simple until you try to do something slightly complex. Then you find yourself nested three levels deep in boolean queries, wondering why your results are wrong, and questioning whether full-text search was worth the trouble.

<!-- more -->

## What You're Actually Querying

Before writing queries, it helps to understand what OpenSearch is doing. It's not a database with rows and columns. It's an inverted index that maps terms to documents. When you index a document, OpenSearch analyzes the text, breaks it into tokens, and builds structures that make searching fast.

This matters because queries behave differently than SQL. Searching for "running" might match "run" depending on your analyzer. Capitalization usually doesn't matter. And relevance scoring means results come back ranked, not in insertion order.

AWS OpenSearch is Amazon's fork of Elasticsearch, created after licensing drama. The query syntax is nearly identical, so most Elasticsearch documentation applies. But AWS adds authentication, VPC integration, and managed scaling that change how you connect and deploy.

## The Two Query Contexts

OpenSearch has two fundamentally different ways to evaluate queries: query context and filter context.

Query context asks "how well does this document match?" and calculates a relevance score. Use this when you care about ranking, like searching for text that should match closely.

Filter context asks "does this document match?" and returns yes or no. Use this for exact matches, date ranges, or any criteria where scoring doesn't matter. Filters are faster and cacheable.

Most real queries mix both. You filter down to relevant documents, then score within that subset.

## Basic Match Queries

The simplest useful query is a match query. It analyzes your search text the same way the indexed text was analyzed, then finds documents containing those terms.

```json
{
  "query": {
    "match": {
      "content": "database optimization"
    }
  }
}
```

This finds documents where the content field contains "database" or "optimization" or both. Documents with both terms score higher. The analyzer might also match "databases" or "optimizing" depending on your mapping.

If you want all terms to be present, use the operator parameter:

```json
{
  "query": {
    "match": {
      "content": {
        "query": "database optimization",
        "operator": "and"
      }
    }
  }
}
```

Now only documents containing both terms match. This is stricter but often more useful.

## Exact Matches With Term Queries

Sometimes you need exact matching without analysis. Term queries match the exact value in the inverted index:

```json
{
  "query": {
    "term": {
      "status": "published"
    }
  }
}
```

This is perfect for enums, IDs, or any field you mapped as a keyword. But it's a common trap: if you use a term query on an analyzed field, it probably won't match what you expect. The indexed value was lowercased and tokenized, but your query term wasn't.

For multiple exact values, use terms (plural):

```json
{
  "query": {
    "terms": {
      "status": ["published", "featured"]
    }
  }
}
```

This matches documents where status is either value.

## Combining Queries With Bool

Real searches rarely use a single condition. Bool queries let you combine multiple queries with boolean logic:

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "content": "elasticsearch" } }
      ],
      "filter": [
        { "term": { "status": "published" } },
        { "range": { "date": { "gte": "2025-01-01" } } }
      ],
      "should": [
        { "match": { "tags": "tutorial" } }
      ],
      "must_not": [
        { "term": { "archived": true } }
      ]
    }
  }
}
```

The clauses mean different things:

Must clauses are required and affect scoring. The document must match, and how well it matches influences relevance.

Filter clauses are required but don't affect scoring. Use these for exact matches and ranges. They're faster and cacheable.

Should clauses are optional but affect scoring. If a document matches, its score increases. With minimum_should_match, you can require some number of should clauses.

Must_not clauses exclude documents. They run in filter context, so they're fast.

Understanding when to use must versus filter is the difference between slow queries that produce weird results and fast queries that work correctly.

## Range Queries for Numbers and Dates

Range queries handle numeric ranges, date ranges, and anything that has ordering:

```json
{
  "query": {
    "range": {
      "price": {
        "gte": 10,
        "lt": 100
      }
    }
  }
}
```

The operators are gte (greater than or equal), gt (greater than), lte (less than or equal), and lt (less than). For dates, OpenSearch understands ISO 8601 format and relative expressions:

```json
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "now-7d/d",
        "lt": "now/d"
      }
    }
  }
}
```

This matches documents from the last seven days. The /d rounds to the start of the day. You can use h for hours, m for minutes, and other units.

Range queries should always go in filter context unless you need scoring based on how close a value is to the range.

## Phrase Matching for Exact Order

Match queries find documents containing your terms in any order. Match_phrase requires them in the exact order:

```json
{
  "query": {
    "match_phrase": {
      "content": "machine learning"
    }
  }
}
```

This only matches documents where "machine" appears immediately before "learning". It won't match "learning about machines" or "machine-based learning techniques".

You can allow some flexibility with slop:

```json
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "machine learning",
        "slop": 2
      }
    }
  }
}
```

Now "machine deep learning" would match because there's only one word between the terms.

## Multi-Field Searches

Searching across multiple fields is common. The multi_match query makes this easier:

```json
{
  "query": {
    "multi_match": {
      "query": "database optimization",
      "fields": ["title^3", "content", "tags^2"]
    }
  }
}
```

The caret notation boosts field importance. Matches in title count three times more than matches in content. This lets you tune relevance without complex bool queries.

The type parameter changes how multi_match works. The default is best_fields, which scores documents based on the single best matching field. Using most_fields scores across all fields, and cross_fields treats all fields as one big field, useful for searches that might span field boundaries.

## Wildcard and Prefix Queries

Sometimes you need partial matching. Prefix queries match terms starting with a prefix:

```json
{
  "query": {
    "prefix": {
      "username": "john"
    }
  }
}
```

This matches "john", "johnny", "johnson", etc. It's okay for small datasets but slow on large ones because it can't use the index efficiently.

Wildcard queries are more flexible but even slower:

```json
{
  "query": {
    "wildcard": {
      "filename": "*.pdf"
    }
  }
}
```

Use these sparingly. If you find yourself using them often, reconsider your mapping. Maybe you need a different analyzer or a separate keyword field.

## Nested Queries for Complex Objects

If your documents contain arrays of objects, you need nested queries. Standard queries don't preserve the relationship between fields in the same object.

Suppose you have documents with this structure:

```json
{
  "comments": [
    { "author": "alice", "text": "great post" },
    { "author": "bob", "text": "thanks alice" }
  ]
}
```

A regular query for author=alice and text="thanks" would incorrectly match this document because both values exist, just in different objects. A nested query preserves the object boundary:

```json
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "match": { "comments.author": "alice" } },
            { "match": { "comments.text": "thanks" } }
          ]
        }
      }
    }
  }
}
```

This only matches if the same comment object has both conditions. For this to work, the mapping must declare comments as a nested type, not just an object.

## Aggregations for Analytics

Queries return documents, but aggregations compute analytics across results. They're like GROUP BY in SQL but more powerful:

```json
{
  "query": {
    "match": { "status": "published" }
  },
  "aggs": {
    "popular_tags": {
      "terms": {
        "field": "tags",
        "size": 10
      }
    }
  }
}
```

This returns the top 10 most common tags across published documents. You can nest aggregations to build complex analytics:

```json
{
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": {
          "avg": { "field": "price" }
        }
      }
    }
  }
}
```

This groups by category and calculates the average price within each category. Aggregations can get arbitrarily complex, computing percentiles, histograms, date ranges, and geographic bounds.

## Practical Patterns

Here are patterns I use constantly:

For autocomplete, use a prefix query on a keyword field or an edge_ngram analyzer. Prefix queries are simpler but edge_ngram is faster at scale.

For faceted search, combine a query with terms aggregations on facet fields. Let users filter by clicking facet values.

For fuzzy matching, use the fuzziness parameter in match queries. This handles typos and spelling variations:

```json
{
  "query": {
    "match": {
      "title": {
        "query": "opensrch",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

For highlighting matches, add a highlight section:

```json
{
  "query": { "match": { "content": "search" } },
  "highlight": {
    "fields": {
      "content": {}
    }
  }
}
```

Results include fragments with matched terms wrapped in emphasis tags.

## Common Mistakes

Using term queries on text fields is the most common error. Text fields are analyzed, so the indexed tokens won't match your search terms. Use match queries for text, term queries for keywords.

Forgetting to paginate leads to performance problems. Always use from and size to limit results:

```json
{
  "from": 0,
  "size": 20,
  "query": { ... }
}
```

For deep pagination, use search_after instead, which is more efficient.

Putting everything in must when filter would work wastes resources. Filters are faster and cacheable. Only use must when you need scoring.

Not setting reasonable timeouts causes hanging queries. Use the timeout parameter:

```json
{
  "timeout": "5s",
  "query": { ... }
}
```

Queries that exceed the timeout return partial results instead of failing completely.

## Testing and Debugging

The explain API shows why a document matched and how its score was calculated:

```json
GET /my-index/_explain/document-id
{
  "query": { ... }
}
```

The response is verbose but invaluable when debugging relevance issues.

The validate API checks query syntax without executing it:

```json
GET /my-index/_validate/query?explain=true
{
  "query": { ... }
}
```

This shows the actual Lucene query OpenSearch constructs, which helps when behavior seems wrong.

The profile API shows where time is spent:

```json
{
  "profile": true,
  "query": { ... }
}
```

Use this to identify slow parts of complex queries.

## Working With AWS OpenSearch

AWS adds authentication via IAM or Cognito. The easiest approach uses IAM signing with the AWS SDK:

```python
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

credentials = boto3.Session().get_credentials()
auth = AWS4Auth(
    credentials.access_key,
    credentials.secret_key,
    'us-east-1',
    'es',
    session_token=credentials.token
)

client = OpenSearch(
    hosts=[{'host': 'your-domain.us-east-1.es.amazonaws.com', 'port': 443}],
    http_auth=auth,
    use_ssl=True,
    verify_certs=True,
    connection_class=RequestsHttpConnection
)
```

This handles credential rotation and temporary credentials automatically.

For VPC-based clusters, you need to run queries from within the VPC or set up a proxy. Direct internet access requires either a public domain or an SSH tunnel.

## When OpenSearch Isn't the Answer

OpenSearch is powerful but not always the right choice. It's overkill for simple filtering on a few fields. A database with proper indexes often works better.

It's not a primary data store. Durability and consistency aren't as strong as a real database. Index your data from a source of truth, don't use OpenSearch as the source.

It struggles with joins. If your queries need complex relational logic, denormalize your documents or reconsider whether search is the right approach.

And it requires tuning. Out-of-the-box settings rarely work well at scale. Plan to adjust shard counts, replica counts, analyzer settings, and refresh intervals based on your workload.

## Making It Work

OpenSearch query syntax has depth that takes time to internalize. Start with simple match and filter queries. Add complexity only when you need it. Test queries against real data and use the explain API when results seem wrong.

The learning curve is real, but once you understand query context versus filter context and when to use which query type, most patterns become straightforward. Your searches will be fast, relevant, and actually return what users expect.
