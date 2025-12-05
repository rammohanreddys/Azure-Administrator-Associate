# Azure DevOps:

The major concepts for CI/CD automation in Azure DevOps primarily revolve around Azure Pipelines and the integration of the other core Azure DevOps services.

**Azure Pipelines Core Concepts:**

Azure Pipelines is the service within Azure DevOps that enables CI/CD automation. The workflow is defined using a Pipeline, which is structured by several key concepts:

|Concept|Description|
|---------:|-----------|
|Pipeline  | The automated workflow that defines the CI/CD process (build, test, deploy). It can be configured using YAML (stored in the repository for Infrastructure as Code practices) or the Classic Editor (GUI).|
|Stages    | A major division within a pipeline, representing a large, logical unit of work (e.g., Build, Test, Staging Deployment, Production Deployment). Stages run sequentially by default.|
|Jobs      | A series of steps that run sequentially inside a stage. Jobs are the execution boundary and typically run on an Agent. A stage can contain multiple jobs, which can run in parallel.|
|Steps     | The smallest building block of a pipeline. A step is either a Task (pre-defined script/action) or a Script (inline command-line, PowerShell, or Bash).|
|Agents    | The computing infrastructure (virtual machine or container) that executes the jobs in the pipeline. They can be: * Microsoft-hosted agents (managed by Azure) * Self-hosted agents (managed by your team, useful for specific dependencies or on-premises deployments).|
|Artifacts | The output of a CI pipeline (e.g., compiled code, executables, Docker images, web application packages). This artifact is consumed by the CD pipeline or a subsequent stage for deployment.|
|Triggers  | Define what starts the pipeline run. Common triggers include: * Continuous Integration (CI) Trigger: Starts the pipeline on every code push to a specific branch (e.g., main). * Pull Request (PR) Trigger: Starts a validation build when a PR is created or updated. * Scheduled Trigger: Starts the pipeline at a specific time.|

```
# ---------------------------------------------------------------------
### STAGE 1: Continuous Integration (CI) - Build and Test
# ---------------------------------------------------------------------

trigger:
- main # 1. CI Trigger: Start the pipeline automatically on any push to the 'main' branch

# Define variables used throughout the pipeline
variables:
  vmImage: 'ubuntu-latest' # The agent image to use for building
  artifactName: 'drop'      # The name of the artifact to publish
  webAppName: 'my-staging-web-app' # The name of the Azure App Service

stages:
- stage: Build
  displayName: 'Build and Test Application'
  jobs:
  - job: BuildJob
    displayName: 'Node.js Build and Artifact Creation'
    pool:
      vmImage: $(vmImage) # Use the Ubuntu agent

    steps:
    # Task 1: Use the correct Node.js version
    - task: NodeTool@0
      inputs:
        versionSpec: '18.x'
      displayName: 'Install Node.js 18.x'

    # Task 2: Install dependencies
    - script: |
        npm install
      displayName: 'Install Dependencies'

    # Task 3: Run unit tests (essential CI step)
    - script: |
        npm test
      displayName: 'Run Unit Tests'

    # Task 4: Build the application for deployment (e.g., compile/bundle)
    - script: |
        npm run build
      displayName: 'Build Application'

    # Task 5: Publish the build output as an artifact
    - task: PublishBuildArtifacts@1
      inputs:
        pathToPublish: 'dist' # Folder containing the deployable output
        artifactName: $(artifactName)
      displayName: 'Publish Build Artifact'

# ---------------------------------------------------------------------
### STAGE 2: Continuous Delivery (CD) - Deploy to Staging Environment
# ---------------------------------------------------------------------

- stage: DeployStaging
  displayName: 'Deploy to Staging'
  dependsOn: Build # Ensures this stage only runs after the 'Build' stage completes successfully
  condition: succeeded() # Only run if the previous stage succeeded

  jobs:
  - deployment: DeployToStaging
    displayName: 'Staging Deployment'

    #### Specifies the environment for deployment, enabling checks/approvals
    environment: 'MyWebApp-Staging'
    pool:
      vmImage: $(vmImage)

    #### Strategy defines how the deployment is executed (runOnce is standard)
    strategy:
      runOnce:
        deploy:
          steps:
          # Step 1: Download the artifact created in the 'Build' stage
          - download: current
            artifact: $(artifactName)
            displayName: 'Download Artifact'

          # Step 2: Deploy the web application using a service connection
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure App Service'
            inputs:
              # Service Connection: Links Azure DevOps to your Azure subscription
              azureSubscription: 'My Azure Subscription Service Connection'
              appType: 'webAppLinux'
              # The name of the Web App in Azure
              appName: $(webAppName)
              # Path to the deployed package (downloaded artifact)
              package: '$(Pipeline.Workspace)/$(artifactName)/**.zip'
              # Ensure deployment logs are detailed
              enableDeploymentTelemetry: true
## ------------------------------------------------------------------------------------------------------------------------- ##
```
**Detailed Explanation of Key YAML Concepts:**

The pipeline is hierarchical: Pipeline --> Stages --> Jobs --> Steps (Tasks or Scripts). This structure allows you to visually separate your CI (Build) from your CD (Deploy).

* **stages:** The main containers. Our example has a Build stage and a DeployStaging stage.
* **jobs:** The execution units. A job runs on a single Agent.
* **steps:** The actual work executed by the agent.
   - **task:** Name@Version: Calls a pre-built task from the Azure DevOps Marketplace (e.g., NodeTool@0, PublishBuildArtifacts@1).
   - **script:** Executes a command-line script (e.g., npm install).

**CI Automation Essentials:**

* **trigger:** - main: This is the heart of Continuous Integration. It ensures that every time a developer merges code into the main branch, the pipeline starts automatically.
* **task:** PublishBuildArtifacts@1: Once the build and tests are successful, this task takes the output folder (dist) and makes it available to subsequent stages, usually by uploading it to an internal storage location. This output is the Artifact.

**CD Automation Essentials:**

* **dependsOn:** Build: This enforces the sequence. The DeployStaging stage will wait for the Build stage to finish and will not proceed if the build fails.
* **environment:** 'MyWebApp-Staging': This is critical for Continuous Delivery.
   - It links the deployment to a logical Environment in Azure DevOps.
   - It allows you to configure Approvals and Checks (e.g., manual approval from a manager) before the deployment job starts, which is a core governance feature.
* **strategy:** runOnce: Defines the deployment pattern. runOnce means it executes the steps one time. Other strategies exist for rolling or blue/green deployments.
* **task:** AzureWebApp@1: This is the deployment task. It uses a pre-configured Service Connection (azureSubscription) to authenticate with your Azure account and deploy the application package to the specified Web App (appName).

## Triggers:

1. **CI trigger:** automatically starts a pipeline run whenever a code change is pushed to a specified branch in your repository.

```
**Example:**
### azure-pipelines.yml

trigger:
  branches:
    include:
      - main
      - releases/*
    exclude:
      - features/experimental/* # Exclude branches for experimental features
  paths:
    include:
      - src/app/** # Trigger only if files under src/app/ change
    exclude:
      - docs/* # Do not trigger if only files under docs/ change
```
2. **PR Trigger:** A Pull Request (PR) trigger in an Azure DevOps pipeline is a configuration that automatically runs your pipeline whenever a pull request is created or updated targeting a specific branch. This is essential for implementing Branch Policies and ensuring that code changes are built and tested before they are merged into the target branch.

```
# azure-pipelines.yml

pr:
  branches:
    include:
      - main
      - feature/*
    exclude:
      - features/experimental/* # Do not run for PRs targeting experimental branches
  paths:
    include:
      - src/web/** # Only run the pipeline if files in the web app source code change
    exclude:
      - docs/* # Skip the run if only documentation changes
```

3. **Scheduled Trigger:** A Scheduled Trigger in an Azure DevOps pipeline allows you to run your pipeline automatically at specific times and days, independent of code changes. This is typically used for nightly builds, weekly reports, or maintenance tasks.

```
# azure-pipelines.yml

schedules:
- cron: "0 0 * * *" # Run at 00:00 (midnight) every day
  displayName: Daily Midnight Build
  branches:
    include:
      - main # Only run this schedule against the 'main' branch
  always: true
```

4. **Resource Based Triggers:** A Resource-Based Trigger in an Azure DevOps YAML pipeline automatically starts a pipeline run when an event occurs in an external or internal resource that the pipeline is configured to consume. This is the core mechanism for creating multi-stage or multi-pipeline Continuous Delivery (CD) workflows.

The primary types of resources you can use to trigger a pipeline are:
* **Pipelines** (Pipeline Completion Trigger): Triggers one pipeline upon the successful completion of another.
* **Repositories** (Multi-repo Trigger): Triggers the pipeline when code is pushed to a repository other than the one containing the pipeline YAML file.
* **Containers:** Triggers the pipeline when a new version of a Docker image is pushed to a container registry.
* **Packages:** Triggers the pipeline when a new version of a NuGet or npm package is published.
* **Webhooks:** Triggers the pipeline using a generic webhook (e.g., from an external service).

**Pipeline Completion Trigger:**

The most common resource trigger is the Pipeline Completion Trigger, which allows you to chain pipelines together. For example, a "Build" pipeline completes and automatically triggers a "Deployment" pipeline.

```
# CD-Pipeline.yml

# 1. Define the resource (the pipeline that runs first)
resources:
  pipelines:
  - pipeline: ci_app      # Alias for the resource. Use this name for variables and artifacts.
    source: CI-Pipeline   # The actual name of the source pipeline in Azure DevOps.
    project: MyProject    # Optional: Project where the source pipeline is located (defaults to current project).
    trigger: 
      branches:
        include:
        - main           # The CD-Pipeline will only trigger if CI-Pipeline completes on 'main'.
      tags:
      - production       # Optional: You can also require specific tags on the source pipeline run.

# 2. Define the jobs/stages that will run when the trigger fires
stages:
- stage: Deploy
  displayName: Deploy to Environment
  jobs:
  - job: DeploymentJob
    steps:
    - script: echo "The CI-Pipeline completed! Starting deployment with artifacts from CI-Pipeline."
    - task: DownloadPipelineArtifact@2
      inputs:
        artifact: drop
        pipeline: ci_app # Use the resource alias to download artifacts from the completed run
```

**Repository Resource Trigger (Multi-Repo):**

You can also trigger a pipeline when a change is made to a different Git repository defined as a resource. This is useful when one pipeline depends on code from multiple repositories.

```
# Pipeline A.yml (This pipeline runs when repo B changes)

resources:
  repositories:
  - repository: secondary_repo # Alias for the repository resource
    type: git                  # Type of repository (git, github, bitbucket, etc.)
    name: AnotherProject/RepoB # Name of the repo (Project/RepoName)
    ref: main                  # Optional: specify a ref (branch)
    trigger:
      branches:
        include:
        - main                # Trigger on pushes to the 'main' branch of RepoB
```
