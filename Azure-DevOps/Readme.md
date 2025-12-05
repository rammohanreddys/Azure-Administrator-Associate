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
