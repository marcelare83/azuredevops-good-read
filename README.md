# Lessons learned - Azure Pipelines, Code Coverage, .NET, SonarQube

## ⏩ Performance-related tips

<details>
  <summary>
    <h4> Always run the pipeline on a Linux-based agent </h4>
  </summary>

When possible, always run the pipeline on a Linux-based agent instead of a Windows-based one. In my experience this can reduce the runtime by up to 50%, depending on the pipeline workload:

```yaml
pool:
  vmImage: "ubuntu-latest"
```

(`ubuntu-latest` is also the default Agent image in Azure DevOps, so if you don't specify anything else this will be used)

This in itself means you should avoid using the [VSBuild@1](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vsbuild-v1?view=azure-pipelines) and [VSTest@2](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vstest-v2?view=azure-pipelines) tasks as these can only be run on a Windows-based agent. You should instead use the [DotNetCoreCLI@2](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines) task for building/restoring/testing .NET code.

</details>

<details>
  <summary>
    <h4> Run multiple jobs in parallel </h4>
  </summary>

A great way to reduce the total time a build takes is to run multiple smaller jobs in parallel instead of one big job. A `job` in Azure DevOps will automatically run on a separate agent, and thus run in parallel, as opposed to a separate step or task that will run sequentially on the same agent. In the following example, Job A & Job B will run at the same time:

```yaml
jobs:
- job: A
  steps:
  - bash: echo "A"

- job: B
  steps:
  - bash: echo "B"
```

 [You can find more info about how Jobs work here](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml).

This means you can for example have one job that builds the main source code and another job that runs the tests and a third job that does some kind of analysis, and they can all run simultaneously.

You can combine this with something like a a final "publish" step that utilizes the [`dependsOn`](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/conditions?view=azure-devops&tabs=yaml%2Cstages#use-the-output-variable-from-a-job-in-a-condition-in-a-subsequent-job) parameter to make sure it doesn't run until all the other jobs have finished successfully.

The only thing that limits parallelization in this way is more or less if there are dependencies between steps of a specific pipeline that cannot be run in a different job (which runs on a different agent). Keep in mind thought that you can utilize the [PublishPipelineArtifact](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines) and [DownloadPipelineArtifact](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/download-pipeline-artifact-v2?view=azure-pipelines) tasks to publish some kind of result from one job and download it in another.

It is also easier to run more jobs in parallel if your .NET code is using .NET Core (.NET 6/7/8) and not .NET Framework because it is possible to build individual .NET Core projects in the pipeline without having to build everything in a solution, which is not the case for .NET Framework*. This means that if you have let's say `Project1.csproj` and `TestProject1.csproj` that are both part of `MySolution.sln`, you can create two jobs that builds that specific `.csproj` file and not the entire solution, and then run them in parallel.

>\* technically it is possible to build individual projects instead of a entire solution using the [VSBuild@1](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vsbuild-v1?view=azure-pipelines) too, by pointing to the path of a `.csproj` file using the ["solution"](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vsbuild-v1?view=azure-pipelines#:~:text=solution%20%2D-,Solution,-string.%20Required.%20Default) property. The issue is that this takes an extreme amount of time, often as long as building the entire solution in my experience. I suspect this is because the legacy "VS" tasks are built from the ground-up to work using a solution file, so this way of building is a bit of a hack and does not seem to be officially supported.

Keep in mind that as you increase the number of parallel jobs that are being run you might start getting into issues where there are no available agents because all of them are already busy with other jobs. In this case you can go to Project Settings > Pipelines > Parallel jobs and increase the number there (if you're willing to pay for it). This costs [$40 per month per additional Microsoft-hosted agent](https://azure.microsoft.com/en-us/pricing/details/devops/azure-devops-services/).

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/6df852e3-f12e-4e79-9072-c2858490edeb)

</details>

<details>
  <summary>
    <h4> Limit frequency of static code analysis runs </h4>
  </summary>

If you are using some kind of tool for static code analysis, such as [SonarQube](https://www.sonarsource.com/products/sonarqube/), keep in mind that doing this on a medium to large solution adds a significant amount of time to the build process as well as taking time to run the actual analysis (at least when it comes to SonarQube). Therefore a good way to save time is to reduce this analysis when it is not "required" (based on preferences and/or organizational policies).

One way to achieve this is to create a script like this (this is a PowerShell example):

```ps
# SonarQube analysis will be run if any of these are true:
# 1. The runSonarQube parameter is manually set to true.
# 2. The build is for either of these branches: dev, master, Releases/*.
# 3. The build is for a pull request to either of these branches: dev, master, Releases/*.

Param(
    [string]$runSonarQubeParameter,
    [string]$buildSourceBranch,
    [string]$pullRequestTargetBranch
)

$branchRequiresAnalysis = $buildSourceBranch -eq 'dev' -or $buildSourceBranch -eq 'master' -or $buildSourceBranch -like 'Releases/*'
$prTargetRequiresAnalysis = $pullRequestTargetBranch -eq 'dev' -or $pullRequestTargetBranch -eq 'master' -or $pullRequestTargetBranch -like 'Releases/*'

if ($branchRequiresAnalysis) {
    Write-Host "Branch requires SonarQube analysis."
}

if ($prTargetRequiresAnalysis) {
    Write-Host "Pull request target requires SonarQube analysis."
}

if ($runSonarQubeParameter -eq "True") {
    Write-Host "SonarQube analysis is manually requested."
}

$sonarQubeShouldBeRun = $runSonarQubeParameter -eq "True" -or $branchRequiresAnalysis -or $prTargetRequiresAnalysis

if (!$sonarQubeShouldBeRun) {
    Write-Host "##[warning] NOTE: SonarQube analysis will be skipped for this build!"
}

# Set "SonarQubeShouldBeRun" variable to be used in rest of the pipeline
# See: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-variables-scripts?view=azure-devops&tabs=powershell
Write-Host "##vso[task.setvariable variable=sonarQubeShouldBeRun]$sonarQubeShouldBeRun"
```
This script can then be called like this:

```yaml
# This script will set the variable "sonarQubeShouldBeRun" to true or false
- task: PowerShell@2
  displayName: Determine if SonarQube analysis should be run
  inputs:
    targetType: filePath
    filePath: build/scripts/Determine-SonarQubeAnalysisShouldBeRun.ps1
    arguments: "${{ parameters.runSonarQube }} '$(Build.SourceBranchName)' '$(System.PullRequest.TargetBranchName)'"
```

and the `sonarQubeShouldBeRun` variable can be used to control the analysis steps like this:

```yaml
- task: SonarQubeAnalyze@5
  displayName: "SonarQube: Run analysis"
  condition: and(succeeded(), eq(variables.sonarQubeShouldBeRun, true))
```

This will make sure that the analysis is only run for the main branches that require this analysis as well as any pull requests that target those branches. On top of this it also takes into account a manual flag "runSonarQubeParameter" that can be used when an analysis run is required outside of these situations.

This parameter can be set like this:

```yaml
parameters:
  - name: runSonarQube
    type: boolean
    default: false
    displayName: Run SonarQube analysis
```

and will show up like this in the UI when queueing a new pipeline build:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/92a7569e-bc34-4b2f-bd08-8a19269d7289)

</details>

<details>
  <summary>
    <h4> Avoid unnecessary .NET project building due to implicit restore & build </h4>
  </summary>

Make sure you are not accidentally building a project/solution multiple times - Because of the way that the [implicit restore](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build#implicit-restore) works for dotnet tasks it is very easy to, say, first build a solution with a project and a test project in one step, and then run the tests using the `DotNetCoreCLI@2` task, not knowing that this will trigger an additional unnecessary build of that test project. 

A way to get around this is to either (a) skip the first build step and simply run the test task as this will also build and restore the project, or (b) keep the separate build task and then call the test task with the `--no-build` argument:

```yaml
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

</details>

<details>
  <summary>
    <h4> Avoid the "PublishCodeCoverageResults@1" task due to poor performance </h4>
  </summary>

The [`PublishCodeCoverageResults@1`](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-code-coverage-results-v1?view=azure-pipelines) task in Azure DevOps is used to take already produced code coverage results (JaCoCo / Cobertura format) and publish it to the pipeline. This makes the code coverage results show up as a tab in the pipeline run summary in Azure DevOps:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/e806df44-f98d-4d44-805b-9d3c1c256a30)

The issue is that this task is so incredibly slow that it basically makes it unusable unless the amount of files is very small. This is a known issue and has [been reported years ago](https://github.com/microsoft/azure-pipelines-tasks/issues/4945) but not fixed (yet).

An alternative to this stand-alone task you can use if you are running a .NET test task is to specify that code coverage should be collected and published during the test run, like this:

```yaml
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

Combining both these things could look like this:

```yaml
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

</details>
  
## ⏩ Code coverage-related tips

<details>
  <summary>
    <h4> How to enable collecting of code coverage during test execution </h4>
  </summary>

- DotNetCoreCLI@2

```yaml
- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/MyTestProject.csproj"
    arguments: >
      --collect "Code Coverage" # <----
```

Note that you can specify the argument like this `--collect "Code Coverage;Format=Xml"` to collect the coverage information in an XML format instead of the binary `.coverage` format.

- VSTest@2

```yaml
- task: VSTest@2
  displayName: "🔬 VS Test"
  inputs:
    testAssemblyVer2: |
      Tests/**/MyTestProject.dll
      !**/obj/**
    platform: "AnyCPU"
    configuration: "Release"
    codeCoverageEnabled: true # <----
```

</details>

<details>
  <summary>
    <h4> How to produce code coverage results from parallel jobs </h4>
  </summary>

Even though it requires some extra work, it _is_ possible to collect code coverage from multiple parallel jobs, which allows you to significantly improve performance for large solutions with long build times and many tests (see [performance-related tips](#performance-related-tips)).

- Run all tests in multiple jobs, in each job you need to checkout the code, build the relevant code and then run the tests, make sure you specify that code coverage should be collected. You then need to publish the test results (`.coverage` and `.trx` files) so we can download them in the job that is going to run the actual SonarQube analysis:

```yaml
  ############################
  #### RUN TESTS
  ############################

  # The `--no-build` flag will skip building the test project before running it (since we already built in the previous step)
  # It also implicitly sets the --no-restore flag
  # https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-test
  - task: DotNetCoreCLI@2
    displayName: "🔬 dotnet test"
    inputs:
      command: "test"
      projects: "${{ parameters.testsToRun }}"
      testRunTitle: "${{ parameters.testSuite }}"
      publishTestResults: true
      arguments: >
        --configuration Release
        --collect "Code Coverage"
        --no-build

  ############################
  #### PUBLISH ARTIFACT
  ############################

  # Copy relevant files to a "TestResults" folder
  # so we can publish them as an artifact without including the entire TempDirectory
  # NOTE: We only look for .coverage files one sub-directory down, because there exists
  # duplicates of these files further down, which we do not want to copy.
  - task: CopyFiles@2
    displayName: "Copy test result files to $(Agent.TempDirectory)/TestResults"
    inputs:
      SourceFolder: "$(Agent.TempDirectory)"
      Contents: |
        **/*.trx
        */*.coverage
      TargetFolder: "$(Agent.TempDirectory)/TestResults"
      flattenFolders: true

  - task: PublishPipelineArtifact@1
    displayName: "Publish pipeline artifact: ${{ parameters.jobName }}"
    inputs:
      targetPath: "$(Agent.TempDirectory)/TestResults"
      artifactName: ${{ parameters.jobName }}
```

- You will also need to have one job that prepares the analysis, builds the source code and runs the analysis. It is _not_ possible to split the actual analysis and the preparation/building (more info [here](#-unable-to-run-the-sonarqubeprepare5-and-sonarqubeanalyze5-tasks-in-different-jobs)).
- In the preparation step you will need to specify the path where SonarQube can find the test result files (`.trx`) and the code coverage files (will be converted from `.coverage` to `.xml`):
```yaml
- task: SonarQubePrepare@5
  displayName: "SonarQube: Prepare"
  inputs:
    SonarQube: "SonarQube"
    scannerMode: "MSBuild"
    projectKey: "PROJECT_KEY"
    projectName: "PROJECT_NAME"
    extraProperties: |
      sonar.cs.vscoveragexml.reportsPaths=$(Agent.TempDirectory)/TestResults/merged.coverage.xml
      sonar.cs.vstest.reportsPaths=$(Agent.TempDirectory)/TestResults/*/*.trx
```

- After we build the source code but before performing the SonarQube analysis we need to download all the test run result artifacts that were published using the `PublishPipelineArtifact@1` task:
```yaml
- task: DownloadPipelineArtifact@2
  displayName: "Download test run artifacts"
  inputs:
    targetPath: $(Agent.TempDirectory)/TestResults
```
**Note that if the tests take longer to run than building the project then the pipeline will attempt to download the test run artifacts before they are available which will result in the results in SonarQube being incorrect. You can potentially solve this with a delay (or in some more sophisticated way using a script and Azure CLI)**:

```yaml
- task: PowerShell@2
  displayName: "⏳ Delay for 2 minutes to wait for test result artifacts to be available"
  inputs:
    targetType: inline
    script: "Start-Sleep -Seconds 120"

- task: DownloadPipelineArtifact@2
  displayName: "Download test run artifacts"
  inputs:
    targetPath: $(Agent.TempDirectory)/TestResults
```

- We then convert the code coverage files to XML and merge them into one big file using the [`dotnet-coverage` tool](https://learn.microsoft.com/en-us/dotnet/core/additional-tools/dotnet-coverage) (note that we output the final result in the path that we specified in the SonarQube preparation task):

```yaml
- task: PowerShell@2
  displayName: "Install the 'dotnet-coverage' tool"
  inputs:
    targetType: inline
    script: dotnet tool install dotnet-coverage --global --ignore-failed-sources

- script: >
    dotnet-coverage merge -o $(Agent.TempDirectory)/TestResults/merged.coverage.xml -f xml -r $(Agent.TempDirectory)/TestResults/*.coverage --remove-input-files
  displayName: "Merge and re-format code coverage files to XML"
```

- The final step we have to do is fix the path information in the code coverage files. This is because when the files are generated, they retain information about the relative path to the source file that is being tested:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/117f0948-a4d6-40a8-9e8b-c19832a412e4)

Because these files were generated on different agents, the part of the path information that refers to the agent (`C:\agent2\_work`) will not be the same as the path for the source files on the agent we downloaded the results to. We have to fix this manually, otherwise SonarQube won't recognize the coverage information as valid. We can do that like this:

```yaml
- task: PowerShell@2
  displayName: "Fix code coverage file paths in merged.coverage.xml"
  inputs:
    targetType: filePath
    filePath: build/scripts/Fix-CodeCoverageFilePaths.ps1
    arguments: -pathToCoverageFile "$(Agent.TempDirectory)/TestResults/merged.coverage.xml"
  condition: and(succeeded(), eq(variables.sonarQubeShouldBeRun, true))
```

The script this task uses looks like this:

```powershell
# When generating code coverage reports, the paths to the source files are stored in the code coverage file.
# Because we run multiple test jobs on different agents in parallel, it leads to the code coverage file paths being different on each agent.
# This script fixes the paths in the code coverage file so that they are correct from the point-of-view of the agent running the code coverage analysis.

Param(
    [string]$pathToCoverageFile
)

Write-Host "Fixing file paths in code coverage file '$pathToCoverageFile'..."

$localPath = (Get-Location).Path
$linuxFilePattern = '/home/vsts/work/\d+/s'
$onPremWindowsFilePattern = 'C:\\agent\\_work\\\d+\\s'
(Get-Content -path $pathToCoverageFile -Raw) -replace $linuxFilePattern, $localPath -replace $onPremWindowsFilePattern, $localPath | Set-Content $pathToCoverageFile
```

- After this we can finally run the analysis step:

```yaml
- task: SonarQubeAnalyze@5
  displayName: "SonarQube: Run analysis"
```

- We should then get both code coverage information and test results into SonarQube:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/70972161-7cc6-4a6a-891f-c1f168df0565)

</details>

<details>
  <summary>
    <h4> Code coverage results in PRs in Azure DevOps </h4>
  </summary>
  
[There is support](https://learn.microsoft.com/en-us/azure/devops/pipelines/test/codecoverage-for-pullrequests?view=azure-devops) for showing code coverage information for Pull Requests in Azure DevOps, if you have it enabled it shows up like this:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/7326a09a-04f3-4eee-b685-262ca603e032)

To enable this you need to:
1. Add build validation for the target branch so that a new build is run and checked when a new PR is opened
2. In your build pipeline you need to enable gathering of code coverage* from your test runs and publish the results then you will "automatically" get code coverage information in the PR as shown above:

    ```yaml
    - task: DotNetCoreCLI@2
      displayName: "🔬 dotnet test"
      inputs:
        command: "test"
        projects: "**/MyTestProject.csproj"
        publishTestResults: true # <----
        arguments: >
          --collect "Code Coverage" # <----
    ```
    
\* **Note that only the binary `.coverage` format is [currently supported](https://learn.microsoft.com/en-us/azure/devops/pipelines/test/codecoverage-for-pullrequests?view=azure-devops#which-coverage-tools-and-result-formats-can-be-used-for-validating-code-coverage-in-pull-requests), so you need to make sure you are publishing this format** 

</details>
  
## ⏩ Various "gotchas" to watch out for

<details>
  <summary>
    <h4> Running "dotnet tool install" on Linux </h4>
  </summary>
  
If you run the `dotnet tool install` command in a task on an agent running on a Linux-based OS w/ a project that has multiple project files you might run into issues where the task fails to complete with an error message along the lines of the folder containing multiple project files. I think this is related to the fact that .NET Core CLI will automatically restore any .NET projects in the working directory and does not like if there are multiple of them. The process of installing a new dotnet tool does not require this to happen, so it is ostensibly a bug, but I might be missing something.

One way to get around this issue is to set the ["workingDirectory" parameter](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/powershell-v2?view=azure-pipelines#:~:text=workingDirectory%20%2D-,Working%20Directory,-string.) to an arbitrary folder in the repository that does **not** contain any `.csproj` files at all:

```yaml
- task: PowerShell@2
  displayName: "Install the 'dotnet-coverage' tool"
  inputs:
    targetType: inline
    workingDirectory: "$(Build.SourcesDirectory)/ArbitraryFolder" # <----
    script: dotnet tool install dotnet-coverage --global --ignore-failed-sources
```

</details>

<details>
  <summary>
    <h4> The "PublishPipelineArtifact" task doesn't flatten folders </h4>
  </summary>

If you have a need to publish and download artifacts between different jobs in a pipeline you can use the [PublishPipelineArtifact](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines) and [DownloadPipelineArtifact](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/download-pipeline-artifact-v2?view=azure-pipelines) tasks in Azure DevOps. One thing to keep in mind when doing this is that the "PublishPipelineArtifact" task doesn't flatten folders, i.e. if you download an artifact "MyCoolArtifact1" & "MyCoolArtifact2" with some arbitrary files into "MyFolder", then it will result in the files being put into `MyFolder/MyCoolArtifact1` and `MyFolder/MyCoolArtifact2` instead of directly into `MyFolder/...`.

One way solve this is to first download the pipeline artifacts and then use the ["CopyFiles@2"](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/copy-files-v2?view=azure-pipelines&tabs=yaml) task w/ the `flattenFolders` parameter set to `true`:

```yaml
- task: CopyFiles@2
  displayName: "Copy test result files to $(Agent.TempDirectory)/TestResults"
  inputs:
    SourceFolder: "MyFolder"
    Contents: "**"
    TargetFolder: "MyFolder"
    flattenFolders: true # <----
```

`Contents: "**"` copies all files in the specified source folder and all files in all sub-folders. Note that this is the default value, I've explicitly added it here for the sake of clarity.

</details>

<details>
  <summary>
    <h4> "CopyFiles" doesn't work as expected when trying to copy multiple specific file types </h4>
  </summary>
  
If you want to use the ["CopyFiles@2"](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/copy-files-v2?view=azure-pipelines&tabs=yaml) task to copy specific file types (like `.xml`, `.coverage`, `.trx`) instead of all files in a specific folder, you need to make sure that you do not write it over multiple lines using single quotes, like this:

```yaml
- task: CopyFiles@2
  inputs:
    SourceFolder: "$(Build.SourcesDirectory)"
    Contents: |
      '**\bin\**\*.dacpac'
      '**\PublishProfile\*.publish.xml'
    TargetFolder: "$(Build.ArtifactStagingDirectory)"
```

You instead need to write it like this (without quotes), otherwise the files won't be found:

```yaml
- task: CopyFiles@2
  inputs:
    SourceFolder: "$(Build.SourcesDirectory)"
    Contents: |
      **\bin\**\*.dacpac
      **\PublishProfile\*.publish.xml
    TargetFolder: "$(Build.ArtifactStagingDirectory)"
```

More information about this bug can be found [in this forum post](https://stackoverflow.com/a/70874760).

</details>

<details>
  <summary>
    <h4> Installing new software on a self-hosted agent could require a restart before it takes effect </h4>
  </summary>
  
If you are running your pipeline on a self-hosted agent and have tasks that install new software, for example using `dotnet tool install`, then a restart of the agent could be required for it to recognize this new tool/software.

> I noticed this when I tried installing the `dotnet-coverage` tool and it said it was already installed but at the same time when trying to use it in a task it said it wasn't installed, leading to a catch-22. Restarting the agent solved this issue.

The solution was taken from [this forum post](https://stackoverflow.com/a/62712205).

</details>

<details>
  <summary>
    <h4> Default test result folder location for "DotNetCoreCLI@2" vs. "VSTest@2" </h4>
  </summary>

Note that the "DotNetCoreCLI@2" task puts test results in `$(Agent.TempDirectory)` whereas the legacy "VSTest@2" task puts it in `$(Agent.TempDirectory)/TestResults`.

This location can be re-configured for the "VSTest@2" using the [`resultsFolder` parameter](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vstest-v2?view=azure-pipelines#:~:text=resultsFolder%20%2D-,Test%20results%20folder,-string.%20Default%20value).

</details>

## ⏩ SonarQube-related tips

<details>
  <summary>
    <h4> Unable to run the "SonarQubePrepare@5" and "SonarQubeAnalyze@5" tasks in different jobs </h4>
  </summary>

There are two main SonarQube-related tasks available in Azure DevOps:
- [SonarQubePrepare@5](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/sonar-qube-prepare-v5?view=azure-pipelines)
- [SonarQubeAnalyze@5](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/sonar-qube-analyze-v5?view=azure-pipelines)

From what I've read, seen, and tried, these two tasks **HAVE** to be run in the same job, otherwise the analysis step fails. I don't know specifically what the prepare step does and there isn't a lot of documentation about that either, but there is some kind of magic that happens behind the scenes. All I know is that part of what happens is that it creates a hidden `.sonarqube` folder in the working directory with a bunch of files. I even tried copying the entire working folder to another agent after the prepare step and then running analyze and that still didn't work...

What this means is that you cannot optimize the pipeline to do something along the lines of running one job that prepares the analysis and builds the source code in parallel with test jobs and then end with a job that run the SonarQube analysis. Instead you need to either run everything in one big job, or have one job that prepares the analysis, builds the source code, and then waits to download the test results from separate jobs before running the analysis (this leads to timing issues etc.).

_Either way, it is annoying..._

</details>

<details>
  <summary>
    <h4> Getting unit test results into SonarQube </h4>
  </summary>

This is configured through the ["Test execution parameters"](https://docs.sonarsource.com/sonarqube/9.9/analyzing-sources-code/test-coverage/test-execution-parameters/) in SonarQube and specified in the "SonarQubePrepare@5" task.

For C# it could look like this:

```yaml
- task: SonarQubePrepare@5
  displayName: "SonarQube: Prepare"
  inputs:
    SonarQube: "SonarQube"
    scannerMode: "MSBuild"
    projectKey: "${{ parameters.projectName }}"
    projectName: "${{ parameters.projectName }}"
    extraProperties: |
      sonar.cs.vstest.reportsPaths=$(Agent.TempDirectory)/*.trx # <---- 
```

If you are using XUnit or NUnit instead of VSTest/MSTest there are [alternative report paths](https://docs.sonarsource.com/sonarqube/9.9/analyzing-source-code/test-coverage/test-execution-parameters/#csharp) for these.

This test result report is what makes this information show up in SonarQube:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/3bb97775-d70a-4f61-980f-1370dc550006)

</details>

<details>
  <summary>
    <h4> Be mindful of supported code coverage formats in SonarQube </h4>
  </summary>
  
Keep in mind that SonarQube only supports [certain code coverage formats for certain languages](https://docs.sonarsource.com/sonarqube/9.9/analyzing-source-code/test-coverage/test-coverage-parameters/).

For example: The "Cobertura" code coverage format is not supported for `C#`, but it is supported for `Flex` and `Python`. This can become confusing because "Cobertura" shows up as a popular code coverage format in a lot of C# articles etc. So even though it is fully possible to generate this format "out-of-the-box" for C# code, SonarQube won't see it as valid.

Also, the binary `.coverage` format that is generated by default when collecting code coverage info in .NET is **not** supported by SonarQube, but at the same time this is the expected format when publishing test results to the Azure DevOps pipeline. Therefore it is recommended to collect this data in the binary format and then re-format it into a XML format that is compatible w/ SonarQube before running the analysis step.

</details>

<details>
  <summary>
    <h4> SonarQube + .NET + Windows-based agent = Magic? </h4>
  </summary>

If you are analyzing .NET code using SonarQube and are using a Windows-based agent, then there seems to be some convention-based magic happening behind the scenes that is good to know about.

This is what is [written in SonarQube's documentation](https://docs.sonarsource.com/sonarqube/9.9/analyzing-source-code/test-coverage/dotnet-test-coverage/#visual-studio-code-coverage):
> "[...] when you are using an Azure DevOps Windows image for your build. In these cases, the .NET Framework scanner will automatically find the coverage output generated by the --collect "Code Coverage" parameter without the need for an explicit report path setting. It will also automatically convert the generated report to XML. No further configuration is required."

So the paths to the test results is implicitly set (it relies on them being in `$(Agent.TempDirectory)/TestResults` and after that checks a few other "reasonable" places) AND the binary `.coverage` format is automatically converted to XML. In my opinion this way of doing it involves way too much "magic" and is just needlessly confusing if you are not using this exact setup... 

Either way, if you are not running on a Windows image (which you [should avoid for performance reasons](#-always-run-the-pipeline-on-a-linux-based-agent-)) then you need to do this yourself instead.

Converting `.coverage` to XML (this was also refenced in [this chapter](#-avoid-the-publishcodecoverageresults1-task-due-to-poor-performance-):

```yaml
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

Specifying test result paths:

```yaml
- task: SonarQubePrepare@5
  displayName: "SonarQube: Prepare"
  inputs:
    SonarQube: "SonarQube"
    scannerMode: "MSBuild"
    projectKey: "PROJECT_KEY"
    projectName: "PROJECT_NAME"
    extraProperties: |
      sonar.cs.vscoveragexml.reportsPaths=$(Agent.TempDirectory)/TestResults/.coverage.xml # <---- 
      sonar.cs.vstest.reportsPaths=$(Agent.TempDirectory)/TestResults/*/*.trx # <---- 
```

</details>

## ⏩ Azure Pipeline-related tips

<details>
  <summary>
    <h4> Azure DevOps pipeline templates </h4>
  </summary>

You can utilize [templates](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-includes) in Azure DevOps to define reusable content, logic, and parameters in YAML pipelines.

The way this works is that you can first define some kind of YAML code in one repository, say `TemplateRepository` in the `Infrastructure` project in Azure DevOps:

TemplateRepository/templates/mytemplate.yml:
```yaml
parameters:
  - name: message
    type: string

steps:
  - bash: echo ${{ parameters.message }}
```

You can then use that template like this (you specify the template repository as a resource and then you point to the file you want to use):

```yaml
resources:
  repositories:
    - repository: infrastructure # variable name
      type: git
      name: Infrastructure/TemplateRepository # Project/Repo
      ref: refs/heads/main # branch

trigger: none

pool:
  vmImage: ubuntu-latest

steps:
  - template: templates/mytemplate.yml@infrastructure
    parameters:
      message: "My cool message"
```

</details>

<details>
  <summary>
    <h4> Conditions for pipeline templates </h4>
  </summary>

There is no support for the [`condition` keyword](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/conditions) when using templates, meaning you cannot write something like this:

```yaml
  - template: templates/mytemplate.yml@infrastructure
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
```

What you CAN do though is define the condition like this:

```yaml
  - ${{ if ne(variables['Build.Reason'], 'PullRequest') }}:
      - template: templates/jobs/publish-viedoc-package.yml
```

and that will work.

</details>

<details>
  <summary>
    <h4> Working directory when checking out multiple repositories </h4>
  </summary>
  
If you are using pipeline templates and want to for example use script files from the repository that the template YAML file is checked into, you can accomplish this by [checking out multiple repositories](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/multi-repo-checkout?view=azure-devops) in the pipeline. So you are both checking out the repository that is using the template repository as a resource and the template repository itself:

TemplateRepository/templates/mytemplate.yml:

```yaml
steps:
  # checkout: self is implicitly defined for all pipelines, I've adde it here for clarity
  - checkout: self

  # checkout the infrastructure repo so we can run script files from it
  - checkout: infrastructure
```

One important thing to keep in mind when doing this is that it will change the way the default working directory works. Normally when you checkout a repository the root of that repository will be the working directory, so if you have a repostitory `MyRepo` which has a folder `MyFolder` with a textfile `Test.txt`, then in that pipeline you can use the path `MyFolder/Test.txt` to find that file. 

But if you are checking out more than one repo then the working directory will be one folder "up", with the root folder of all repositories being inside that folder. For example, you checkout `MyRepo` and `MyRepo2`. The path to the `Test.txt` in `MyRepo` will now be `MyRepo/MyFolder/Test.txt` instead of just `MyFolder/Test.txt` like it was before:

```
- MyRepo
  - MyFolder
    - Test.txt

- MyRepo2
  - MyFolder2
    - Test2.txt
```

</details>

<details>
  <summary>
    <h4> Running tests in pipeline that require "Azurite" </h4>
  </summary>

If you are running tests that require a local "Azurite" instance, for example for emulating Azure Storage, then you need a way to duplicate this functionality when running these tests in your CI pipeline.

One way to do that is to add this task:

```yaml
# Azurite is required for some tests to run as expected
# See: https://learn.microsoft.com/en-us/samples/azure-samples/automated-testing-with-azurite/automated-testing-with-azure/
- bash: |
    npm install -g azurite
    mkdir azurite
    azurite --silent --location azurite &
  displayName: "Install and Run Azurite"
```
</details>

## ⏩ .NET-related tips

<details>
  <summary>
    <h4> Setting "testRunTitle" when running the "DotNetCoreCLI@2" or "VSTest@2" task </h4>
  </summary>

You can customize the value of the `testRunTitle` parameter for both the [DotNetCoreCLI@2](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines#:~:text=testRunTitle%20%2D-,Test%20run%20title,-string.%20Optional.%20Use) task and the [VSTest@2](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/vstest-v2?view=azure-pipelines#:~:text=testRunTitle%20%2D-,Test%20run%20title,-string.) task.

For example:

```yaml
- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/MyTestProject.csproj"
    testRunTitle: "Application (Clinic)"
```

This will make the results that show up in the test results tab in Azure DevOps more readable and clear:

![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/202e8372-0e13-4636-814f-d7581b171dec)
![image](https://github.com/OscarBennich/lessons-learned-azure-devops-sq-dotnet/assets/26872957/b8f36dd0-3de1-434b-af30-14117e3db34f)

</details>

<details>
  <summary>
    <h4> Issues related to specifying a local NuGet feed in `nuget.config` </h4>
  </summary>
  
If you have specified a local NuGet feed in a `nuget.config` file in the root of your repository, like this:

```xml
<?xml version="1.0" encoding="utf-8"?>

<configuration>
  <packageSources>
    <clear />
    <add key="Local" value="%USERPROFILE%\.viedoc\local\packages\nuget" /> # <----
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

you will run into issues whenever you try to build any project in this repository in a pipeline. This is because of the implementation of the [implicit restore](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build#implicit-restore) that triggers whenever you build a `.csproj`. This restore step will [default to using the feed information provided by the `nuget.config` file](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-restore#specify-feeds).

What this leads to is that whenever you try to trigger a task that builds or runs tests using your .NET projects, e.g.:

```yaml
- task: DotNetCoreCLI@2
  displayName: "🏗 dotnet build"
  inputs:
    command: "build"
    projects: "**/ProjectToBuild.csproj"

- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/ProjectToTest.csproj"
```

then the implicit restore will kick in and try to find the local feed specified in the `nuget.config` file, which it is unable to do, so the task fails.

One way to solve this is to separate the `restore`, `build` and `test` steps. This allows you to specify what feed should be used in the `restore` step, and then specify that no restore should be performed in the subsequent steps.

We specify the organization feed to use using the [`vstsFeed` parameter](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines#:~:text=vstsFeed%20%2D-,Use%20packages%20from%20this%20Azure%20Artifacts%20feed,-Input%20alias%3A):

```yaml
- task: DotNetCoreCLI@2
  displayName: "♻ dotnet restore"
  inputs:
    command: "restore"
    projects: "**/MyProject.csproj"
    vstsFeed: "ProjectName/FeedName" # <----
```

We then use the `--no-restore` argument to skip the implicit restore:

```yaml
- task: DotNetCoreCLI@2
  displayName: "🏗 dotnet build"
  inputs:
    command: "build"
    projects: "**/MyProject.csproj"
    arguments: >
      --no-restore # <----
```

We also use the `--no-build` argument when running the tests:

```yaml
# The `--no-build` flag will skip building the test project before running it (since we already built in the previous step)
# It also implicitly sets the --no-restore flag
- task: DotNetCoreCLI@2
  displayName: "🔬 dotnet test"
  inputs:
    command: "test"
    projects: "**/MyProject.csproj"
    arguments: >
      --no-build # <----
```

- [More info](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines#why-is-my-build-publish-or-test-step-failing-to-restore-packages)

</details>

<details>
  <summary>
    <h4> Running tests after building a solution with both .NET Core & .NET Framework-based test projects </h4>
  </summary>

If you are running a pipeline that is building a solution with a mix of .NET Core & .NET Framework projects then you can run into issues if you run the [`VSTest` task](https://learn.microsoft.com/sv-se/azure/devops/pipelines/tasks/reference/vstest-v2?view=azure-pipelines) after that. 

This seems to be because the task gets "confused" about what test adapter to use during this run. A way to solve this is to utilize the `pathtoCustomTestAdapters` property and point to one of the .NET Framework projects in the solution (it doesn't matter which one):

```yaml
- task: VSTest@2
  displayName: "🔬 VS Test"
  inputs:
    testAssemblyVer2: |
      Tests/**/MyTestProject.dll
      !**/obj/**
    platform: "AnyCPU"
    configuration: "Release"
    pathtoCustomTestAdapters: "Tests/MyTestProject/bin/Release/net472/ # <----
```
</details>
