# Lessons learned / Tips & Tricks related to using Azure DevOps pipelines w/ .NET & SonarQube

## Gotchas
### `dotnet tool`
- dotnet tool install gotcha for Linux regarding "workingDirectory"
- NOTE: We need to set the workingDirectory to an arbitrary folder (e.g. "Scripts")
- that does not have any .NET projects in it, because the .NET Core CLI will
- automatically restore any .NET projects in the working directory.

### "PublishPipelineArtifact" task
- PublishPipelineArtifact@1 doesn't flatten folders
- i.e. if you download an artifact "MyCoolArtifact1" &  "MyCoolArtifact2" with some arbitrary files into "MyFolder", then it will result in MyFolder/MyCoolArtifact1/... & MyFolder/MyCoolArtifact1/...
- We can potentially solve this by using the "CopyFiles@2" task w/ the `flattenFolders` parameter set to true

### Installing new software on a self-hosted agent
- Installing new software on an self-hosted agent (like a dotnet tool) during a build could lead to you needing to restart the agent
- See: https://stackoverflow.com/a/62712205

## Performance
### "PublishCodeCoverageResults" task
- The "PublishCodeCoverageResults@1" is incredibly slow...
  - https://github.com/microsoft/azure-pipelines-tasks/issues/4945
  - Basically makes it unusable.
  - But publishing directly from the "DotNetCoreCLI@2" task is much, much faster (I don't know why this is)

## Code coverage
### Code coverage results in PRs in Azure DevOps
- https://learn.microsoft.com/en-us/azure/devops/pipelines/test/codecoverage-for-pullrequests?view=azure-devops
- https://learn.microsoft.com/en-us/azure/devops/pipelines/test/codecoverage-for-pullrequests?view=azure-devops#which-coverage-tools-and-result-formats-can-be-used-for-validating-code-coverage-in-pull-requests

## SonarQube

## Local NuGet feed

## .NET
### Implicit restore & build
- https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build#implicit-restore

### "pathtoCustomTestAdapters"
- "pathtoCustomTestAdapters" for VSTest task:
- Required when building a solution with both .NET Core & .NET Framework-based test projects
- Use the "pathtoCustomTestAdapters" property to point to one of the .NET Framework-projects, like so:
- pathtoCustomTestAdapters: "Tests/Internal/Viedoc.Worker.Tests/bin/Release/net472/"

## Pipeline templates

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
