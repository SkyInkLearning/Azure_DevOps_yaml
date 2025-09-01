# Purpose:

This is the yaml code that I am using in the DevOps course. It was created with the help of chatgpt to get me started on the project. After having gone through and learned about what each section does, I have made this summary to be able to refer back to it in the future. What is shown might not be the exact version of the code that I had at the end.


## Best explanation:

I think my best way of explaning a yaml pipeline is that you are explaining to a virtual machine what to install and run and what things you want to trigger the VM to do it. 

So you define which packages are needed and define key value pairs, under variables, and use scripts to tell the VM which CLI commands to run to properly check that your code and tests pass before packaging it up for source control and deployment.

### Triggers:

```yaml
trigger:
  branches:
    include:
      - develop
      - master
```

The pipeline will run everytime you merge anything into the chosen trigger branches.


### PR:

```yaml
pr:
  branches:
    include:
      - develop
      - master
```

The "pr" section is telling the pipeline that if a pull request is targets either of these branches, run the pipeline. It will run the pipeline on the code in the pull request, making sure that the tests clear on all the new code in the pull request.

### Pool:

```yaml
pool:
  vmImage: 'windows-latest'
```

This is an agent pool which picks a Microsoft-hosted windows agent. 

An agent is a virtual machine, which has common tools pre-installed, that runs the pipeline steps. So in the pool above you request a virtual machine which has windows latest things on it to spin up your code and run through the steps, like the tests, to check so everything works. After it's done, the virtual machine will reset itself.

You can also utilize a self hosted agent. 

### Variables:

```yaml
variables:
  buildConfiguration: Release
  solution: CarSimulatorApp.sln
  webProject: CarSimulator.Server/CarSimulator.Server.csproj
```

The variable section is just as in normal code. You pre-set key value pairs so that you can use them later on in the yaml. For example the buildConfiguration above gets used in alot of the scripts below. 

### Stages:

```yaml
stages:
```

It's just the different stages that the virtual machine will go through. 

### Build:

```yaml
  - stage: Build
    displayName: 'Build Stage'
    jobs:
      - job: BuildJob
        displayName: 'Building car simulator app solution.'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: 6.0.x
            displayName: 'Install .NET 6 SDK'

          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: 9.0.x
            displayName: 'Install .NET 9 SDK'

          - script: dotnet restore $(solution)
            displayName: 'Restoring NuGet packages.'
          
          - script: dotnet build $(solution) --configuration $(buildConfiguration) --no-restore
            displayName: 'Building solution in Release mode.'

```

The build stage is going to install any nuget packages that are in the solution and run a build of the code.

### Test:

```yaml
  - stage: Test
    displayName: 'Testing Stage'
    dependsOn: Build
    jobs:
      - job: TestJob
        displayName: 'Running tests.'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: 6.0.x
            displayName: 'Install .NET 6 SDK'

          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: 9.0.x
            displayName: 'Install .NET 9 SDK'

          - script: dotnet test $(solution) --configuration $(buildConfiguration) --no-build --verbosity normal
            displayName: 'Running all tests.'

```

This is the testing stage which has a dependency on the build stage. If the build stage fails, it wont continue into this stage. 

It uses a script with a CLI command to run all of the tests that are found in the solutions code. 

### Publish:

```yaml
  - stage: Publish
    displayName: 'Publishing Stage'
    dependsOn: Test
    jobs:
      - job: PublishJob
        displayName: 'Publishing artifacts.'
        steps:
          - script: dotnet publish $(webProject) --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/publish_output
            displayName: 'Publishing web app to staging directory.'

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '$(Build.ArtifactStagingDirectory)/publish_output'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/car-sim.zip'
              replaceExistingArchive: true
            displayName: 'Create deployment zip'

          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/car-sim.zip'
              artifact: 'drop'
            displayName: 'Uploading pipeline artifact.'

```

The publish stage depends on the success of the test stage. 

Artifacts, that are created in this stage, are the outputs from the build. It can be binaries, configs, a published app or for example a zip file.

In the case of the above pipeline it will create a zip file, an artifact, of the code at that stage which will saved for later and used for deploying the code to the website/app. It can also be used to easily be able to go back to an older version if something goes wrong in the future as the artifacts created are saved in azure devops. 

### Deploy:

```yaml
  - stage: Deploy
    displayName: 'Deploying Stage'
    dependsOn: Publish
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master'))   # 👈 only deploy on master
    jobs:
      - deployment: DeployJob
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadPipelineArtifact@2
                  inputs:
                    artifact: 'drop'
                    path: '$(Pipeline.Workspace)/drop'
                  displayName: 'Download published artifact'

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'DevOpsWebsite'
                    appName: 'carsim'
                    appType: 'webApp'
                    package: '$(Pipeline.Workspace)/drop/car-sim.zip'
                  displayName: 'Deploying car simulator to web app.'

```

The deploy stage depends on the publishing stage. 

It will push the artifact to replace the current version that is on the site. The condition in the yaml makes sure that the pipeline only deploys the code to the website if you merge into master otherwise it will skip this step. 


### Scripts:

```yaml
- script: dotnet publish CarSimulatorApp.sln --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)
      displayName: 'Publish web app to staging directory'
```

Scripts is a way to get the virtual machine to do things. In the script you can add CLI commands while using the variables that you set at the top.

The displayname is what is shown on the UI when the pipeline is being run.

### How to require a pr for a branch:

To force pull requests on certain branches you have to set up branch policies. 
In the repository section, click branches and then go to the specific branches policies and change the following;

Require a minimum number of reviewers, a linked work item, comment resolution. Add a build validation, which requires the PRs to pass the pipeline before merging, and block direct pushes to the branch. You will need a pipeline to be able to set the build to be validated.
