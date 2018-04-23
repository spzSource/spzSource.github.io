---
layout: post
title:  "Setup code coverage for VSTS and .NET Core."
date:   2018-04-22 22:48:00 +0300
categories: VSTS
tags: VSTS, dotnet, code coverage
---

Let's imagine that you have VSTS build pipeline for continuously build and test you project. Moreover your project can also has many (or not) tests.To understand what places in your code are covered and what not, you might want to gather coverage code coverage statistic for your tests. Let's take a look how we can do this using out-of-box solution in VSTS.

First of all there are a few changes that need to be done before:

*   Add `Microsoft.CodeCoverage` package reference to your test projects (this step is not required if the package is bundled into the dotnet SDK);
*   Add `<DebugType>Full</DebugType>` to each of project in the solution.

The option `<DebugType>Full</DebugType>` is required to emit a classic windows PDB symbols instead of symbols in portalble format (`<DebugType>Portable</DebugType>`). This is a temporary workaround for the [issue](https://github.com/Microsoft/vstest/issues/800).

After steps above, you can run code coverage in two ways:

*   using .NET Core SDK (through `dotnet test`);
*   using Visual Studio's component called `vstest.console.exe`.

Both of this commands correctly works only on Windows and require _Visual Studio Enterprise_ to be installed :( See the following [issue](https://github.com/Microsoft/vstest/issues/1312). And according to that [RFC](https://github.com/Microsoft/vstest-docs/blob/master/RFCs/0021-CodeCoverageForNetCore.md), support for Linux and Mac will not be in the near future.

So, let's consider code coverage through `vstest.console.exe` tool in scope of VSTS test task.

![VSTS test task]({{ "/assets/vsts_test_task.png" | absolute_url }})

Here is a basic configuration for _Visual Studio Test_ task:

| Setting Name          | Value                             |
| --------------------- | --------------------------------- |
| Select tests using    | Test assemblies                   |
| Test assemblies       | \*\*\\\*.Tests.dll                |
| Search folder         | $(System.DefaultWorkingDirectory) |
| Settings file         | tests.runsettings.xml             |
| Code coverage enabled | true                              |

![VSTS test task]({{ "/assets/vsts_test_task_configuration.png" | absolute_url }})

**Important:** It is important to specify the scope for code coverage, because by default the tool calculates coverage result across all existing assemblies in the output directory. As a result coverage result might be less that it is.

You can specify coverage scope through `*.runsettings.xml` file like in example below:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="Code Coverage" uri="datacollector://Microsoft/CodeCoverage/2.0">
        <Configuration>
          <CodeCoverage>
            <ModulePaths>
              <Include>
                <ModulePath>.*ims.*\.dll$</ModulePath>
              </Include>
            </ModulePaths>
            <UseVerifiableInstrumentation>True</UseVerifiableInstrumentation>
            <AllowLowIntegrityProcesses>True</AllowLowIntegrityProcesses>
            <CollectFromChildProcesses>True</CollectFromChildProcesses>
            <CollectAspDotNet>False</CollectAspDotNet>
          </CodeCoverage>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

The following section means that we want to include all assemblies containing `ims` section in the assembly name.

```xml
<Include>
    <ModulePath>.*ims.*\.dll$</ModulePath>
</Include>
```

If `<Include>` is empty, then code coverage processing includes all assemblies (.dll and .exe files) that are loaded and for which .pdb files can be found, except for items that match a clause in an <Exclude> list.

For further details you can go through documentation [here](https://msdn.microsoft.com/en-us/library/jj159530.aspx).

That's it. Now it is possible to gather coverage statistics during tests execution. The result should looks like below:

![VSTS test task]({{ "/assets/vsts_test_task_coverage_result.png" | absolute_url }})
