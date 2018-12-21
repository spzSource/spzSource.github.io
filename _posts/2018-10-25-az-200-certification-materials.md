---
layout: post
title: "Azure Developer certification: materials"
date: 2018-10-25 21:30:00 +0300
categories: certification
tags: azure, certification, exams, development, az-200
---

Microsoft recently has [announced](https://www.microsoft.com/en-us/learning/community-blog-post.aspx?BlogId=8&Id=375159) their role-based certification program. Six new job roles were announced: developer, architect, administrator, DevOps, Modern Desktop administrator, and enterprise administrator. But my interest will be aimed against azure developer role.

There are two exams:

- AZ-200 - Microsoft Azure Developer Core Solutions Exam;
- AZ-201 - Microsoft Azure Developer Advanced Solutions Exam;

Learning path for achieving certificate:


![Learning Path]({{ "/assets/azure_certification/learning_path.png" | absolute_url }})

## Exam sections

### Select the appropriate cloud technology solution (15-20%)

---

* Select an appropriate compute solution (_May include but not limited to: Leverage appropriate design patterns; select appropriate network connectivity options; design for hybrid topologies_)
    * Design patterns:
        * [Cloud Design Patterns](https://azureinteractives.azurewebsites.net/CloudDesignPatterns/default.html)
        * [Azure Architecture Patterns](https://docs.microsoft.com/en-us/azure/architecture/)
        * Azure Learn: [Tour Azure services and features](https://docs.microsoft.com/en-gb/learn/modules/tour-azure-services-and-features/)
---

* Select an appropriate integration solution (_May include but not limited to: Address computational bottlenecks, state management, and OS requirements; provide for web hosting if applicable; evaluate minimum number of nodes_)
  * Azure Learn: [Design for efficiency and operations in Azure](https://docs.microsoft.com/en-gb/learn/modules/design-for-efficiency-and-operations-in-azure/)
  * Book: [Cloud Application Architecture Guide](https://azure.microsoft.com/en-us/campaigns/cloud-application-architecture-guide/)
  * Book: [Azure for Architects](https://azure.microsoft.com/en-us/resources/azure-for-architects/)

---

* Select an appropriate storage solution (_May include but not limited to: Validate data storage technology capacity limitations; address durability of data; provide for appropriate throughput of data access; evaluate structure of data storage; provide for data archiving, retention, and compliance_)
  * Azure Learn: [Introduction to Azure Storage](https://docs.microsoft.com/en-gb/learn/modules/intro-to-data-in-azure/)
  * Azure Learn: [Choose a data storage approach in Azure](https://docs.microsoft.com/en-gb/learn/modules/choose-storage-approach-in-azure/)

---

### Develop for cloud storage (30-35%)
* Develop solutions that use storage tables (_May include but not limited to: Connect to storage; design and implement policies to tables; query a table storage by using code_)
  * Azure Learn: [Store data in Azure](https://docs.microsoft.com/en-gb/learn/paths/store-data-in-azure/)
  * Documentation: [Azure Storage Table Design Guide: Designing Scalable and Performant Tables](https://docs.microsoft.com/en-us/azure/cosmos-db/table-storage-design-guide)

---

* Develop solutions that use Cosmos DB storage (_May include but not limited to: Choose a consistency level; choose appropriate API for Cosmos DB Storage; create, read, update, and delete tables in Cosmos storage by using code; manage documents and collections in Cosmos DB Storage_)
  * Azure Learn: [Create an Azure Cosmos DB database built to scale](https://docs.microsoft.com/en-gb/learn/modules/create-cosmos-db-for-scale/)
  * Azure Learn: [Insert and query data in your Azure Cosmos DB database](https://docs.microsoft.com/en-gb/learn/modules/access-data-with-cosmos-db-and-sql-api)
  * Azure Learn: [Distribute your data globally with Azure Cosmos DB](https://docs.microsoft.com/en-gb/learn/modules/distribute-data-globally-with-cosmos-db/)

---

* Develop solutions that use file storage (_May include but not limited to: Implement quotas for File Shares in storage account; move items in file shares between containers asynchronously; set file storage container properties in metadata_)

---

* Develop solutions that use a relational database (_May include but not limited to: Create, read, update, and delete database tables by using code; implement dynamic data masking_)
  * Azure Learn: [Work with relational data in Azure](https://docs.microsoft.com/en-gb/learn/paths/work-with-relational-data-in-azure/)

---

* Develop solutions that use blob storage (_May include but not limited to: Create a shared access signature for a blob; move items in blob storage between containers asynchronously; set blob storage container properties in metadata_)

---

* Developing for caching and content delivery solutions (_May include but not limited to: Develop for Azure Redis cache, storage on Content Delivery Networks (CDNs); develop code to address session state and cache invalidation_)

---

_*Links to these articles will be updated regularly_