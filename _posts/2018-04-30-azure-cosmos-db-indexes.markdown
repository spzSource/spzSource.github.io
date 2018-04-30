---
layout: post
title:  "Cosmos DB: Configure indexing to perform range queries for string values."
date:   2018-04-30 22:10:00 +0300
categories: Azure
tags: Azure, CosmosDB, Indexing
---

By default, the `CosmosDB SDK` serializes `DateTime` values as `ISO 8601` string. 
So, that's a problem if you want to execute ranges or ORDER BY queries based on DateTime properties, 
because by default Cosmos DB engine creates HASH-based indexes for string values. 

See below:

```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {
            "path": "/*",
            "indexes": [
                {
                    "kind": "Range",
                    "dataType": "Number",
                    "precision": -1
                },
                {
                    "kind": "Hash",
                    "dataType": "String",
                    "precision": 3
                }
            ]
        }
    ],
    "excludedPaths": []
}

```

To be able execute range-based queries against datetime values represented as a string need to configure index policies.
Let's go through the example.

```cs
public class Transaction
{
    public Guid Id { get; set; }
    ...
    public DateTime InvoiceDate { get; set; }
}
```

Index configuration must be the following:

```json

{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {
            "path": "/InvoiceDate/?",
            "indexes": [
                {
                    "kind": "Range",
                    "dataType": "String",
                    "precision": -1
                }
            ]
        }
    ],
    "excludedPaths": [
        {
            "path": "/"
        }
    ]
}

```

The section `excludePaths` means that we excludes all properties from indexing and then include `InvoiceDate` for indexing. 
More general is overlapped by more specific. 

This example also shows how we can exclude properties from indexing, 
because by default all properties are considered for building reverse index structure. 
If you want to improve write operations, this approach is a first what you need to apply. 

For detailed information you can go through the next pages:

- [How does Azure Cosmos DB index data?](https://docs.microsoft.com/en-us/azure/cosmos-db/indexing-policies)
- [Performance Tips for Azure DocumentDB - Part 1](https://azure.microsoft.com/ru-ru/blog/performance-tips-for-azure-documentdb-part-1-2/)
- [Performance Tips for Azure DocumentDB - Part 2](https://azure.microsoft.com/en-us/blog/performance-tips-for-azure-documentdb-part-2/)

