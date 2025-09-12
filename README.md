# Purpose:

This is the yaml code that I am using in the DevOps course. It was created with the help of chatgpt and claude to get me started on the project. After having gone through and learned about what each section does, I have made this summary to be able to refer back to it in the future. What is shown might not be the exact version I had at the end.

## Best explanation:

I think my best way of explaning a yaml pipeline is that you are explaining to a virtual machine what to do. 

So you define which packages are needed and define key value pairs, under variables, and use scripts to tell the VM which CLI commands to run to properly check that your code and tests pass before packaging it up for source control and deployment.

### Triggers:

```yaml
trigger:
  branches:
    include:
      - develop
      - master

pr:
  branches:
    include:
      - develop
      - master
```

Both the trigger and pr are telling the pipeline to run when a merge/pr is done to either of those two branches. So when finishing a feature you create a pull request to add it to the dev branch which triggers the pipeline to run. After the pr has ran, it will run again on the feature getting merged into the dev branch. 

The same thing happens for the master/main branch, which makes the total pipeline runs four, just to merge get a new feature to the website.

In the case of the pipeline shown below the deploy stage only happens on the merge to the master branch. 

<img src="https://github.com/user-attachments/assets/2c737f5d-b14a-4514-aa0c-7f18d4c10a57" height="400">

### Pool:

```yaml
pool:
  vmImage: 'windows-latest'
```

This is an agent pool in which we are picking a Microsoft-hosted windows agent. There are linux and mac agents as well.

An agent is a virtual machine, which has common tools pre-installed, that runs the pipeline steps. So in the pool above you request a virtual machine which has windows latest things on it to spin up your code and run through the steps, like the tests, to check so everything works. After it's done, the virtual machine will reset itself.

You could also utilize a self hosted agent. 

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

Stages is where you split things into different stages to enable you to see where things might have gone wrong and to be able to re-run specific stages if any of them fail. 

<img src="https://github.com/user-attachments/assets/03d1dace-5720-497a-9513-5a47148685bb" height="400">

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
              version: 9.0.x
            displayName: 'Install .NET 9 SDK'

          - script: dotnet restore $(solution)
            displayName: 'Restoring NuGet packages.'
          
          - script: dotnet build $(solution) --configuration $(buildConfiguration) --no-restore
            displayName: 'Building solution in Release mode.'
```

The build stage compiles the code and makes sure that the code runs without errors. It also resolves any project references and nuGet dependencies to make sure they are all available. If any errors appear this stage of the pipeline will fail and the rest of it wont run. 

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
              version: 9.0.x
            displayName: 'Install .NET 9 SDK'
            
          - script: dotnet restore $(solution)
            displayName: 'Restoring NuGet packages.' 

          - script: dotnet test Car_Simulator_Tests/Car_Simulator_Tests.csproj --configuration $(buildConfiguration) --verbosity normal --logger trx
            displayName: 'Running all tests.'
          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'VSTest'
              testResultsFiles: '**/*.trx'
            condition: always()
```

This is the testing stage which has a dependency on the build stage. If the build stage fails, it wont continue into this stage. 

When using chatgpt to create this pipeline, it told me to add "--no-build" to the script. It seems like gpt was assuming that the testing stage was going to use the build from the build stage and that it didn't work. Im thinking that it might have worked if the build and test stages were combined into one, but I never tested this as the assignment require them to be separate.

It made it so that none of my tests were actually running even though the pipeline would actually show green checkmarks on the stages. When I noticed the problem I removed that and added the publish test results part after having a chat with Claude instead. You also have to add the "--logger trx" part to the test script for the test results to be published to the pipeline.

The test stage uses a script with a CLI command to run all of the tests that are found in the specified project and it should, now that the tests are actually running, cancel on test failure. 


### Publish:

```yaml
  - stage: Publish
    displayName: 'Publishing Stage'
    dependsOn: Test
    jobs:
      - job: PublishJob
        displayName: 'Publishing artifacts.'
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: 9.0.x
            displayName: 'Install .NET 9 SDK'

          - script: dotnet restore $(webProject)
            displayName: 'Restoring NuGet Packages for web project.'

          - script: dotnet publish $(webProject) --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/publish_output
            displayName: 'Publishing web app to staging directory.'

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '$(Build.ArtifactStagingDirectory)/publish_output'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/car-sim-$(Build.BuildNumber).zip'
              replaceExistingArchive: true
            displayName: 'Create deployment zip'
            
          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/car-sim-$(Build.BuildNumber).zip'
              artifact: 'drop'
            displayName: 'Uploading pipeline artifact.'
```

The publish stage depends on the success of the test stage. 

Artifacts, that are created in this stage, are the outputs from the build. It can be binaries, configs, a published app or for example a zip file.

In the case of the above pipeline it will create a zip file, an artifact, of the code at that stage which will saved for later and used for deploying the code to the website/app. It can also be used to be able to go back to an older version if something goes wrong in the future as the artifacts created are saved. 

<img src="https://github.com/user-attachments/assets/03d1dace-5720-497a-9513-5a47148685bb" height="400">

<img src="https://github.com/user-attachments/assets/6b3643ad-d160-42df-81b2-3b3ff196a1df" height="400">

If you click the artifact shown in the image above, you can access the artifact and download it if needed. 

### Deploy:

```yaml
  - stage: Deploy
    displayName: 'Deploying Stage'
    dependsOn: Publish
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master'))
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
                    package: '$(Pipeline.Workspace)/drop/car-sim-$(Build.BuildNumber).zip'
                  displayName: 'Deploying car simulator to web app.'
```

The deploy stage depends on the publishing stage. 

It will use the artifact to replace the current version that is on the site. The condition in the yaml makes sure that the pipeline only deploys the code to the website when the code gets merged into master otherwise it will skip this step as seen in the image below.

<img src="https://github.com/user-attachments/assets/2c737f5d-b14a-4514-aa0c-7f18d4c10a57" height="400">

### Scripts:

```yaml
- script: dotnet publish CarSimulatorApp.sln --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)
      displayName: 'Publish web app to staging directory'
```

Scripts is what you use to force the virtual machine to do the things you need done. In the script you can add CLI commands while using the variables that you set at the top.

The displayname is what is shown on the UI when the pipeline is being run.

### How to require a pr for a branch:

To force pull requests on certain branches you have to set up branch policies. 
In the repository section, click branches and then go to the specific branches policies and change the following;

Require a minimum number of reviewers, a linked work item, comment resolution. Add a build validation, which requires the PRs to pass the pipeline before merging, and block direct pushes to the branch. You will need a pipeline to be able to set the build to be validated.

