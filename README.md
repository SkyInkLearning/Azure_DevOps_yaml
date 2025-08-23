# Purpose:

This is simply the yaml code that I recently used for one of my courses. Here I will go through each section and explain what they do.

```yaml

```

## Triggers:

```yaml
trigger:
  branches:
    include:
      - develop
      - main
```

The pipeline will run everytime you merge anything into the chosen trigger branches which will build the program and run the tests.


## PR:

```yaml
pr:
  branches:
    include:
      - develop
      - main
```

The "pr" section is telling the pipeline that if a pr is targets either of these branches, run the pipeline. 

## Pool:

```yaml
pool:
  vmImage: 'windows-latest'
```

This is an agent pool which picks a Microsoft-hosted windows agent. 

An agent is a virtual machine that runs the pipeline steps. So when you write the pool above, you request a virtual machine which has windows latest to spin up your code and run it to check so everything works. It will then uninstall everything. 

## Variables:

```yaml
variables:
  buildConfiguration: 'Release'
```

## Stages:

```yaml
stages:
```

### Build:

```yaml
- stage: Build
  displayName: 'Build Stage'
  jobs:
  - job: BuildJob
    displayName: 'Build CarSimulatorApp solution'
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '6.0.x'
      displayName: 'Install .NET 6 SDK'

    - script: dotnet restore CarSimulatorApp.sln
      displayName: 'Restore NuGet packages'

    - script: dotnet build CarSimulatorApp.sln --configuration $(buildConfiguration)
      displayName: 'Build solution in Release mode'
```

### Test:

```yaml
- stage: Test
  displayName: 'Test Stage'
  dependsOn: Build
  jobs:
  - job: TestJob
    displayName: 'Run Unit Tests'
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '6.0.x'
      displayName: 'Install .NET 6 SDK'

    - script: dotnet test CarSimulatorApp.sln --configuration $(buildConfiguration) --no-build --verbosity normal
      displayName: 'Run all unit tests'
```

### Publish:

```yaml
- stage: Publish
  displayName: 'Publish Stage'
  dependsOn: Test
  jobs:
  - job: PublishJob
    displayName: 'Publish Artifacts'
    steps:
    - script: dotnet publish CarSimulatorApp.sln --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)
      displayName: 'Publish web app to staging directory'

    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifact: 'drop'
      displayName: 'Upload pipeline artifact'
```

### Deploy:

```yaml
- stage: Deploy
  displayName: 'Deploy Stage'
  dependsOn: Publish
  condition: succeeded()
  jobs:
  - deployment: DeployWeb
    displayName: 'Deploy to Azure Web App (Dev)'
    environment: 'Dev'   # Track deployments/approvals; create an Environment named "Dev"
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              buildType: 'current'
              artifactName: 'drop'
              targetPath: '$(Pipeline.Workspace)/drop'
            displayName: 'Download build artifact (app.zip)'

          - task: AzureWebApp@1
            inputs:
              azureSubscription: '<your-service-connection>'
              appName: '<your-app-service-name>'
              package: '$(Pipeline.Workspace)/drop/app.zip'
            displayName: 'Deploy app to Azure App Service'
```

## Scripts:

```yaml


```

You can run CLI commands in your pipeline. Like if you need to install packages. 

## How to require a pr for a branch:

To force pull requests on certain branches you have to set up branch policies. 
In the repository section, click branches and then go to the specific branches policies and change the following;

Require a minimum number of reviewers, a linked work item, comment resolution. Add a build validation, which requires the PRs to pass the pipeline before merging, and block direct pushes to the branch. 
