---
layout: post
title:  "Build modular applications"
date:   2018-09-15 04:57:00 +0300
categories: architecture
tags: dotnet architecture patterns IoC module modular applications
---

Have you ever seen projects where all entities, models, domain objects, repositories, services are placed in one place based on class types: all entities are on a single project, repositories are all in a another project and so on? Does it pretty familiar for you? ;) I would say this is worst approach that I can imagine. Why it is bad? Well, there are many reasons...

Lets take a look on the following project structure, where same kind of classes are placed in the same place under the same project:

```
Application.sln
    |
    |-- Entities.csproj
        |
        | - Entity1.cs
        | - Entity2.cs
    |-- Repositories.csproj
        |
        | - Repository1.cs
        | - Repository2.cs
    |-- Services.csproj
        |
        | - Service1.cs
        | - Service2.cs
```

## Why is that bad?

### Reason #1 Hard to control changes across distributes teams

Even during code review, it's hard to get the complete picture about what was changed and how the changes affect other code.

Some of the other teams can use classes which were not created to be reused for some of reason. On other side, modules like a virtual contexts helps to get rid of such situations.

### Reason #2 Hard to support

It hard to determine who should support what classes, because everything is located in a single place. On the contrary in case modules we have a clear names for set of classes, so that helps to assign modules owners. Moreover the module name at the first look gives understanding what functionality a module contains.

### Reason #3 It's not possible for independent versioning

In case modules everything that you need for versioning your module is to define public contracts. From other side, when we talking about none modular structure, versioning is impossible, because classes, which are related to different modules are located in the same project.

### Reason #4 Breaking incapsulation and isolation

In case modules (feature modules) we can hide implementation details inside the module without exposing internal classes outside the module. Modules are represented as black-boxes with exposed public contracts for interaction. This approach helps to reduce coupling beetween separate part of application and further provides ability to change internal implementation of module without breaking consumers (for instance, changing data access layer from SQL Server to CosmosDb). 

In case none modular structure it hard to change something independently, because there are no encapsulation and every class are aware of existence others.

## Modular programming

![Modular application diagram]({{ "/assets/modular_applications/Modular_App.png" | absolute_url }})


### When modular design gives benefits

+ when application have a huge and complex domain area;
+ when there are many distributes teams work on the project;
+ when it is required to achieve isolation between separate parts of application;
+ to make it easy for migrating to microservice architecture in future;

### How modules can be implemented

Here is how basic modular program can be implemented:

![Modular application structure]({{ "/assets/modular_applications/Modular_App_Structure.png" | absolute_url }})

Each module have extension method for `IServiceCollection` contract to be able for append module implementations into IoC container.

```cs
    public static class SecondModule
    {
        public static IServiceCollection AddSecondModule(this IServiceCollection services)
        {
            return services;
        }
    }
```

The entry point of application can be like specified below:

```cs
    class Program
    {
        static void Main(string[] args)
        {
            IServiceCollection services = new ServiceCollection()
                .AddFirstModule()
                .AddSecondModule()
                .AddThirdModule();

            IServiceProvider provider = services.BuildServiceProvider();
        }
    }
```

Sharing classes among modules is an exception rather than a rule, so the fewer common dependencies for modules - the more they are isolated. In some cases following for modular architecture require to violate DRY principles for better incapsulation and isolation. It isn't a bad thing, but this is one that needs to be considered.