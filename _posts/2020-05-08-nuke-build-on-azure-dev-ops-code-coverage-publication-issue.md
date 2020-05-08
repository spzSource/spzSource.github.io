---
layout: post
title: "Nuke.Build + Azure Pipelines: how to publish code coverage artifacts correctly"
date: 2020-05-08 22:56:00 +0300
categories: azure
tags: azure, pipelines, devops, nuke, build, coverage, publication, artifacts
---

On my current project we've started use [Nuke.Build](https://nuke.build/) for buld and deployment pipeline for our services. Througout last few months we did a great job for creating fully operation template for a service, allowing to spin up new services in a 10 minutes. At the same time there are still many issues that we need to address. On this article I want to describe one of the issue, which we've been trying to solve a long time.

## What happened

The issue is related to how Azure DevOps (pipelines) performs artifact publishing. We have a regular build pipeline, containing steps for building, testing, calculating code coverage and publishing all this stuff as an artifact for deployment purposes.

Artifacts publishing target (step) looks like stated below:

```cs
Target PublishArtifacts => _ => _
    .DependsOn(Coverage)
    .Executes(() =>
    {
        // upload binary files
        AzurePipelines?.UploadArtifact(ArtifactDirectory);

        // upload code coverage report
        TestOutputDirectory.GlobFiles("*.xml")
            .ForEach(x => AzurePipelines?.PublishCodeCoverage(
                AzurePipelinesCodeCoverageToolType.Cobertura,
                x,
                CoverageReportDirectory));
    });
```

Everything is loking legal, right?! But code above periodically causes build failures with following odd error:

```
The process cannot access the file '{filename.xml}' because it is being used by another process.
```
For a long time we could not understand what part of the build pipeline might cause such issue, until it occured to me that we need to turn on debug output for build agent :). After that I managed to find a stacktrace:

```
##[debug]System.IO.IOException: The process cannot access the file 'filename.xml' because it is being used by another process.
   at System.IO.FileSystem.RemoveDirectoryRecursive(String fullPath, WIN32_FIND_DATA& findData, Boolean topLevel)
   at System.IO.FileSystem.RemoveDirectory(String fullPath, Boolean recursive)
   at Microsoft.VisualStudio.Services.Agent.Worker.CodeCoverage.PublishCodeCoverageCommand.PublishCodeCoverageAsync(IExecutionContext executionContext, IAsyncCommandContext commandContext, ICodeCoveragePublisher codeCoveragePublisher, IEnumerable`1 coverageData, String project, Guid projectId, Int64 containerId, CancellationToken cancellationToken)
   at Microsoft.VisualStudio.Services.Agent.Worker.AsyncCommandContext.WaitAsync()
   at Microsoft.VisualStudio.Services.Agent.Worker.StepsRunner.RunStepAsync(IStep step, CancellationToken jobCancellationToken)
```

Here is a [link](https://github.com/microsoft/azure-pipelines-agent/blob/master/src/Agent.Worker/CodeCoverage/CodeCoverageCommands.cs#L168) on github, explaining why the issue appears. When calling `PublishCodeCoverage` second parameter points to summary file. Build agent expects only single file and remove directory, containing that file, after file publishing completed. Since `PublishCodeCoverage` is fully asynchronious and we are tyring to call `PublishCodeCoverage` multiple times for the same directory, we run into situation when one operation tries to remove directory, which is use by other operation.

## How to fix

Fix is pretty straightforward - we need to merge all coverage results into a single file (coverage.xml in my case) and then call only single `PublishCodeCoverage`, for instance:

```cs
Target PublishArtifacts => _ => _
    .DependsOn(Coverage)
    .Executes(() =>
    {
        AzurePipelines?.UploadArtifact(ArtifactDirectory);

        AzurePipelines?.PublishCodeCoverage(
            AzurePipelinesCodeCoverageToolType.Cobertura,
            CoverageReportDirectory / "coverage.xml",
            CoverageReportDirectory);
    });
```

## Lessons learned
- use detail debug output to gather more information from logs;
- get familiar with libraries, frameworks, and tools you are using, even look at the code if it's possible.

Happy codding!