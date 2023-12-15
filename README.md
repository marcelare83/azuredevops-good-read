# Lessons learned - Azure DevOps pipelines + .NET + SonarQube

## Performance-related tips
### > Run on Linux-based agents when possible
When possible, always run the pipeline on a Linux-based agent instead of a Windows-based one. In my experience this can reduce the runtime by up to 50%, depending on the pipeline workload:

```
pool:
  vmImage: "ubuntu-latest"
```

("ubuntu-latest" is also the default Agent image in Azure DevOps, so if you don't specify anything else this will be used)

### > Implicit restore & build
Make sure you are not accidentally building a project/solution multiple times - Because of the way that the [implicit restore](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build#implicit-restore) works for dotnet tasks it is very easy to, say, first build a solution with a project and a test project in one step, and then run the tests using the `DotNetCoreCLI@2` task, not knowing that this will trigger an additional unnecessary build of that test project. 

A way to get around this is to either (a) skip the first build step and simply run the test task as this will also build and restore the project, or (b) keep the separate build task and then call the test task with the `--no-build` argument:

```
- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/MyTestProject.csproj"
    arguments: >
      --no-build # <----
```

The `--no-build` flag will skip building the test project before running it, it also implicitly sets the --no-restore flag. 
- See https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-test

### > Avoid the "PublishCodeCoverageResults" task
The [`PublishCodeCoverageResults@1`](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-code-coverage-results-v1?view=azure-pipelines) task in Azure DevOps is used to take already produced code coverage results (JaCoCo / Cobertura format) and publish it to the pipeline. This makes the code coverage results show up as a tab in the pipeline run summary in Azure DevOps:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/e806df44-f98d-4d44-805b-9d3c1c256a30)

The issue is that this task is so incredibly slow that it basically makes it unusable unless the amount of files is very small. This is a known issue and has [been reported years ago](https://github.com/microsoft/azure-pipelines-tasks/issues/4945) but not fixed (yet).

An alternative to this stand-alone task you can use if you are running a .NET test task is to specify that code coverage should be collected and published during the test run, like this:

```
- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/MyTestProject.csproj"
    publishTestResults: true # <----
    arguments: >
      --collect "Code Coverage" # <----
```

Note that the [default value for the "publishTestResults" parameter is `true`](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines#:~:text=publishTestResults%20%2D-,Publish%20test%20results%20and%20code%20coverage,-boolean.%20Optional.%20Use) and can therefore be skipped. I've explicitly added it here for the sake of clarity.

Publishing the test results directly form the "DotNetCoreCLI@2" task like this is **much, much faster** and I don't exactly know why. However, the "built-in" code coverage reporting only handles the binary `.coverage` format (which is what is produced if you don't specify another format in the `--collect` argument). Therefore, if you are instead producing code coverage results with some kind of XML-based format ([using "Coverlet" for example](https://github.com/coverlet-coverage/coverlet?tab=readme-ov-file#usage)), then you need to use the stand-alone publish task.

An alternative to this is to instead produce the code results with the `.coverage` format, publish it, and then in a separate task re-format the results to XML. One way to do is to use the [`dotnet-coverage` tool](https://learn.microsoft.com/en-us/dotnet/core/additional-tools/dotnet-coverage). Specific info about re-formatting and/or merging reports using this tool can be found [here](https://learn.microsoft.com/en-us/dotnet/core/additional-tools/dotnet-coverage#merge-code-coverage-reports).

Bringing both these things together would look like this:

```
- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/MyTestProject.csproj"
    publishTestResults: true
    arguments: >
      --collect "Code Coverage"

- task: PowerShell@2
  displayName: "Install the 'dotnet-coverage' tool"
  inputs:
    targetType: inline
    script: dotnet tool install dotnet-coverage --global --ignore-failed-sources

- script: >
    dotnet-coverage merge -o $(Agent.TempDirectory)/coverage.xml -f xml $(Agent.TempDirectory)/*/*.coverage
  displayName: "Re-format code coverage file(s) to XML"
```

This will result in you being able to take advantage of the faster publishing speed of doing it using the `DotNetCoreCLI@2` task while also being able to output the code coverage results in a more generic format (for SonarQube for example).

## Gotchas
### > Running "dotnet tool install" on Linux
If you run the `dotnet tool install` command in a task on an agent running on a Linux-based OS you might run into issues where the task fails to complete with an error message along the lines of the folder containing multiple project files. I think this is related to the fact that .NET Core CLI will automatically restore any .NET projects in the working directory and does not like if there are multiple of them. The process of installing a new dotnet tool does not require this to happen, so it is ostensibly a bug, but I might be missing something.

One way to get around this issue is to set the ["workingDirectory" parameter](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/powershell-v2?view=azure-pipelines#:~:text=workingDirectory%20%2D-,Working%20Directory,-string.) to an arbitrary folder in the repository that does **not** contain any `.csproj` files at all:

```
- task: PowerShell@2
  displayName: "Install the 'dotnet-coverage' tool"
  inputs:
    targetType: inline
    workingDirectory: "$(Build.SourcesDirectory)/ArbitraryFolder" # <----
    script: dotnet tool install dotnet-coverage --global --ignore-failed-sources
```

### The "PublishPipelineArtifact" task doesn't flatten folders

- i.e. if you download an artifact "MyCoolArtifact1" &  "MyCoolArtifact2" with some arbitrary files into "MyFolder", then it will result in MyFolder/MyCoolArtifact1/... & MyFolder/MyCoolArtifact1/...
- We can potentially solve this by using the "CopyFiles@2" task w/ the `flattenFolders` parameter set to true

### "CopyFiles" task
- CopyFiles@2 task gotcha when trying to copy specific file types using "Contents":
- https://stackoverflow.com/a/70874760

### Installing new software on a self-hosted agent
- Installing new software on an self-hosted agent (like a dotnet tool) during a build could lead to you needing to restart the agent
- See: https://stackoverflow.com/a/62712205

### Local NuGet feed
- no-build, no-restore, vsts-feed

## Code coverage
- "publishTestResults"
- "testRunTitle"

- https://github.com/microsoft/azure-pipelines-tasks/issues/4945

- https://docs.sonarsource.com/sonarqube/9.9/analyzing-source-code/test-coverage/dotnet-test-coverage/#visual-studio-code-coverage
- We only recommend the use of this tool when the build agent has Visual Studio Enterprise installed or when you are using an Azure DevOps Windows image for your build. In these cases, the .NET Framework 4.6 scanner will automatically find the coverage output generated by the --collect "Code Coverage" parameter without the need for an explicit report path setting. It will also automatically convert the generated report to XML. No further configuration is required. Here is an example:

- The "Cobertura" code coverage format doesn't work together with C# for SonarQube...?

- PublishTestResults@2 together w/ Karma (JS) tests
- We need one format for the JUnit test that is compatible w/ Azure DevOps and another that is compatible with SonarQube
- karma-junit-reporter
- karma-sonarqube-unit-reporter

### Code coverage results in PRs in Azure DevOps
- https://learn.microsoft.com/en-us/azure/devops/pipelines/test/codecoverage-for-pullrequests?view=azure-devops
- https://learn.microsoft.com/en-us/azure/devops/pipelines/test/codecoverage-for-pullrequests?view=azure-devops#which-coverage-tools-and-result-formats-can-be-used-for-validating-code-coverage-in-pull-requests

## SonarQube

## .NET
### > Visual Studio-based tasks vs. .NET-based tasks
- https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vsbuild-v1?view=azure-pipelines
- https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vstest-v2?view=azure-pipelines
- https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines

### "pathtoCustomTestAdapters"
- "pathtoCustomTestAdapters" for VSTest task:
- Required when building a solution with both .NET Core & .NET Framework-based test projects
- Use the "pathtoCustomTestAdapters" property to point to one of the .NET Framework-projects, like so:
- pathtoCustomTestAdapters: "Tests/Internal/Viedoc.Worker.Tests/bin/Release/net472/"

## Pipeline templates
### Conditions for templates
```
${{ if ne(variables['Build.Reason'], 'PullRequest') }}
```

## Tests run in pipeline that require "Azurite"
```
      # Azurite is required for some tests to run as expected
      # See: https://learn.microsoft.com/en-us/samples/azure-samples/automated-testing-with-azurite/automated-testing-with-azure/
      - bash: |
          npm install -g azurite
          mkdir azurite
          azurite --silent --location azurite &
        displayName: "Install and Run Azurite"
```
