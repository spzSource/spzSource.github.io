---
layout: post
title: "Cosmos DB: does DISTINCT COUNT really work?"
date: 2019-11-30 22:55:00 +0300
categories: azure
tags: cosmosdb, azure, distinct, count, query.
---

For a while ago during UAT testing for one of our monitoring app we've found strange behaviour in one of cosmos queries. The query performs count against distinct elements returned by inner sub-query.
The query itself looks like below:

```sql
SELECT
    VALUE COUNT(1)
FROM (
    SELECT
        DISTINCT c.InstanceId
    FROM c
    WHERE ARRAY_CONTAINS(c.Status, 'InPipeline', true)
)
```

As you may already noticed, this query was used to get distinct count of InstanceIds with `Status = InPipeline`.

On dev environment everything was working correctly, but on environment with large amount of data, distributed across partitions, the query started to return unpredictable results. So assuming that we decided to rise support ticket to Microsoft. After almost one week of messaging Microsoft engineers had reproduced the issue. Here is an answer:

> This query does retrieve the count of distinct properties for the matched set of documents. However, the query is incorrectly recognized as being distributed and hence we execute it against each partition individually and return the sum of counts from all partition.
> Of cource, the proper distribution of such a query is to retrieve the set of distinct value from all partitions and the apply distinct across all result sets before evaluating the count. We currently do not support distributing such queries, but we should have errord on it rather than returning incorrect results. We will be working on fixing this soon.

The answer sounds reasonable except the one fact we've noticed: when we increase RU/s for a collection the query starts to return a correct result... It doesn't match with explanation provided by Microsoft engineers, as from my knowledge a number of physical partitions doesn't depend on throughput changes.

## TL;TD

The query mentioned above does not work. Cosmos Db engineers have three work items for a near future to improve this situation:

- Be able to identify such queries and return an execution error rather incorrectly execute the query - rough ETA is first quater of 2020.
- Add support for DCOUNT w/ proper query distribution - rough ETA is first quater of 2020.
- Support proper distribution of the query - theу might be able to provide limited support for simple queries, but a complete support would be very difficult to achieve as part of SDK code as it would have dependency on the query engine itself. This has to be delivered as a hosted service, which they currently are working on. The rough ETA for the availability of this service is the second half of 2020.

## Conclusion

It's very funny that such simple thing like distinct count doesn't work on such advertised database. Sometimes it seems to me that all the customers are beta testers :) Moreover it takes a long time to get a clear answer from Microsoft support. Here is a funny example: [link](https://feedback.azure.com/forums/263030-azure-cosmos-db/suggestions/38610298-bug-incorrect-results-returned-from-count-distinc)

Summing up, I would say that you have to think twice before you decide to use cosmos db in your current or next projects.