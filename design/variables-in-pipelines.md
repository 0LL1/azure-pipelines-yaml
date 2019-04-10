# Pipeline variables

To bring Build and Release into one "unified pipelines" model, we need to evolve the variables we expose.
The Build.\* and Release.\* variables no longer make sense.
We also have a unique opportunity to clean up other aspects of the variables we expose.
Though we can make some "breaking changes", we have to give customers, first-party tasks, and Marketplace tasks a clear and easy path forward.

Worth noting: "variables" is an overloaded term.
Azure Pipelines has a notion of "pipeline variables".
Most of these are automatically converted into environment variables on the agent.
The major exception is secrets, which must be manually mapped into the environment.
Also, individual steps can include environment-only variables in the `env` statement.

## Problems
* Build.\* and Release.\* variables aren't meaningful in unified pipelines.
* There are dozens of variables, some which are similar but not quite the same.
We're wary of adding too many more, since we've run into technical limitations before (size of environment block on some systems).
Having similar-but-different variables increases the burden on customers and task authors to understand the system.
* Some tasks operate differently based on whether they're "running in build" vs "running in RM".
Soon, that distinction won't exist.
* Despite the dozens of variables, we're missing some obvious useful ones.

## Solution

A full solution needs to include:
- desired future state
- a path to bring tasks forward without breaking them on older definitions
- a path to bring old definitions forward with minimal pain

Ideally we also take this opportunity to rationalize what variables appear in what scopes.
Current scopes are agent, orchestration time, and available 
- (more?)

### Kinds of existing variables

In the existing set of variables, we found:
- Agent compile-time values
- Agent config-time values
- Agent run-time values
- Source and artifact details
- URIs and identifiers for talking back to Azure Pipelines
- Run control options (under customer control)
- Feature flags (under product team control)

### What are good variables?

Since variables take up space both mentally and in the environment block, we want to be judicious about which ones we include.
Guidance on what makes a "good" variable to include:
- Useful both in expressions (Pipelines-side) and scripts (user/runtime-side)
- _Commonly_ required in an ad-hoc script or a task we ship in the box
- Relevant to detecting or dealing with the environment (e.g. that we're running in Azure Pipelines, that we're running in CI, where on disk the pipeline workspace is rooted)

### Existing variables

#### Agent
| Agent. | Example data | Keep, cut, or rename | Notes |
|--------|--------------|----------------------|-------|
| .BuildDirectory | D:\a\1 | rename | replace with Pipeline.Workspace
| .DeploymentGroupId | 1 | **TODO** | only in deployment group jobs
| .Diagnostic | true | keep | non-existent by default
| .DisableLogPlugin.TestFilePublisherPlugin | true | CUT | replace with feature flag
| .DisableLogPlugin.TestResultLogPlugin | true | CUT | replace with feature flag
| .HomeDirectory | C:\agents\2.148.2 | CUT
| .ID | 3 | keep
| .JobName | Job | CUT | replace with a richer job status mechanism
| .JobStatus | Succeeded | CUT | replace with a richer job status mechanism (completely cut the back-compat lower-cased version "agent.jobstatus") |
| .MachineName | fv-az379 | keep
| .Name | Hosted Agent | keep
| .OS | Windows_NT | keep
| .OSArchitecture | X64 | keep
| .ReleaseDirectory | D:\a\r1\a | CUT | _RM only_
| .RetainDefaultEncoding | false | CUT | feature flag
| .RootDirectory | D:\a | keep
| .ServerOMDirectory | C:\agents\2.148.2\externals\vstsom | CUT
| .TempDirectory | D:\a\\_temp | CUT | use operating system temp construct instead
| .ToolsDirectory | C:/hostedtoolcache/windows | **TODO**
| .Version | 2.148.2 | keep
| .WorkFolder | D:\a | keep

#### Build
| Build. | Example data | Keep, cut, or rename | Notes |
|--------|--------------|----------------------|-------|
| .ArtifactStagingDirectory | D:\a\1\a | CUT
| .BinariesDirectory | D:\a\1\b | CUT
| .BuildID | 1174 | rename | **TODO**
| .BuildNumber | 20190401.7 | rename | **TODO**
| .BuildURI | vstfs:///Build/Build/1174 | CUT
| .Clean | true | CUT | already deprecated
| .ContainerID | 2713905 | **TODO**
| .DefinitionID | 14 | CUT | set for RM with a build artifact |
| .DefinitionName | playground | rename | **TODO**
| .DefinitionVersion | 8 | CUT | used in telemetry, but doesn't seem useful
| .QueuedBy | Microsoft.VisualStudio.Services.TFS | rename | **TODO**
| .QueuedByID | 00000002-0000-8888-8000-000000000000 | rename | **TODO**
| .ProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 | rename | **TODO** set for RM with a build artifact
| .ProjectName | Test1 | rename | **TODO** set for RM with a build artifact
| .Reason | IndividualCI | rename | **TODO**
| .Repository.Clean | False | CUT
| .Repository.Git.SubmoduleCheckout | False | CUT
| .Repository.ID | 80ab696a-644c-497e-84fd-b1d98c470825 | CUT
| .Repository.LocalPath | D:\a\1\s | rename | **TODO**
| .Repository.Name | container | rename | **TODO**
| .Repository.Provider | TfsGit | rename | **TODO**
| .Repository.Tfvc.Workspace | ws_12_8 | rename | **TODO**
| .Repository.URI | https://mattc-demo@dev.azure.com/mattc-demo/Test1/_git/container | rename | **TODO**
| .RequestedFor | Matt Cooper | rename | **TODO**
| .RequestedForEmail | macoope@microsoft.com | rename | **TODO**
| .RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename | **TODO**
| .SourceTfvcShelveset | my_shelveset | rename | **TODO**
| .SourceBranch | refs/heads/master | rename | **TODO**
| .SourceBranchName | master | CUT | actual scenario is replaced by **TODO**
| .SourcesDirectory | D:\a\1\s | CUT
| .SourceVersion | 5d7a52ce5a6e5f3cd1a8d1ee36fd2dd3ae731121 | rename | **TODO**
| .SourceVersionAuthor | Matt Cooper | rename | **TODO**
| .SourceVersionMessage | my commit msg | rename | **TODO**
| .StagingDirectory | D:\a\1\a | CUT
| .TriggeredBy.BuildId | | rename | **TODO**
| .TriggeredBy.DefinitionId | | rename | **TODO**
| .TriggeredBy.DefinitionName | | rename | **TODO**
| .TriggeredBy.BuildNumber | | rename | **TODO**
| .TriggeredBy.ProjectID | | rename | **TODO**
| .Type | Build | **TODO** | set for RM with a build artifact

#### Release
| Release. | Example data | Keep, cut, or rename | Notes |
|----------|--------------|----------------------|-------|
| .Artifacts.{artifact_id}.BuildID | 1174 | rename 
| .Artifacts.{artifact_id}.BuildNumber | 20190401.7 | rename 
| .Artifacts.{artifact_id}.BuildURI | vstfs:///Build/Build/1174 | CUT
| .Artifacts.{artifact_id}.DefinitionID | 14 |  rename 
| .Artifacts.{artifact_id}.DefinitionName | playground |  rename 
| .Artifacts.{artifact_id}.ProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 | rename 
| .Artifacts.{artifact_id}.ProjectName | Test1 | rename 
| .Artifacts.{artifact_id}.PullRequest.TargetBranch | refs/heads/master | rename 
| .Artifacts.{artifact_id}.PullRequest.TargetBranchName | master | rename 
| .Artifacts.{artifact_id}.Repository.ID | 80ab696a-644c-497e-84fd-b1d98c470825 | rename 
| .Artifacts.{artifact_id}.Repository.Name | container | rename 
| .Artifacts.{artifact_id}.Repository.Provider | TfsGit | rename 
| .Artifacts.{artifact_id}.RequestedFor | Matt Cooper | rename 
| .Artifacts.{artifact_id}.RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename 
| .Artifacts.{artifact_id}.SourceBranch | refs/heads/master | rename 
| .Artifacts.{artifact_id}.SourceBranchName | master | rename 
| .Artifacts.{artifact_id}.SourceVersion | 5d7a52ce5a6e5f3cd1a8d1ee36fd2dd3ae731121 | rename 
| .Artifacts.{artifact_id}.Type | `Build`, `Jenkins`, `TeamCity`, `Git`, ... | rename 
| .AttemptNumber | 1 | rename 
| .DefinitionEnvironmentID | 2 | **TODO**
| .DefinitionID | 2 | rename 
| .DefinitionName | My Release Pipeline | rename 
| .DeploymentID | 3 | rename 
| .Deployment.RequestedFor | Matt Cooper | rename 
| .Deployment.RequestedForEmail | *** | rename 
| .Deployment.RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename 
| .Deployment.StartTime | 2019-04-02 12:39:52Z | rename 
| .DeployPhaseID | 3 | rename 
| .EnvironmentID | 5 | CUT
| .EnvironmentName | My First Stage | rename
| .Environments.{stage_name}.Status | InProgress | CUT
| .EnvironmentURI | vstfs:///ReleaseManagement/Environment/5 | CUT
| .PrimaryArtifactSourceAlias | _playground | CUT
| .Reason | Manual | rename
| .ReleaseDescription |  | CUT
| .ReleaseID | 5 | rename
| .ReleaseName | Release-2 | rename
| .ReleaseURI | vstfs:///ReleaseManagement/Release/5 | CUT
| .ReleaseWebURL | https://dev.azure.com/mattc-demo/0807fc91-4393-482d-9e23-defdbb7d0857/_release?releaseId=5&_a=release-summary | CUT
| .RequestedFor | Matt Cooper | rename
| .RequestedForEmail | *** | rename
| .RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | rename
| .SkipArtifactsDownload | False | CUT | replace with feature flag
| .TriggeringArtifact.Alias |  | CUT

#### System
| System. | Example data | Keep, cut, or rename | Notes |
|---------|--------------|----------------------|-------|
| System | `build`, `release` | _note: not System.System -- just `System`_
| .AccessToken | (access token) |
| .ArtifactsDirectory | D:\a\1\a |
| .CollectionID | bb420569-6e91-4163-ab87-1a5b192fd50c |
| .CollectionURI | https://dev.azure.com/mattc-demo/ |
| .Culture | en-US |
| .Debug | false |
| .DefaultWorkingDirectory | D:\a\1\s |
| .DefinitionID | 14 |
| .DefinitionName | playground |
| .EnableAccessToken | `SecretVariable`, `False` |
| .HostType | `build`, `release`, `deployment` |
| .IsScheduled | False |
| .JobAttempt | 1 |
| .JobDisplayName | Job |
| .JobID | 12f1170f-54f2-53f3-20dd-22fc7dff55f9 |
| .JobIdentifier | Job.__default |
| .JobName | __default |
| .JobParallelismTag | Public |
| .JobPositionInPhase | 1 |
| .ParallelExecutionType | None |
| .PhaseDisplayName | Job |
| .PhaseID | 3a3a2a60-14c7-570b-14a4-fa42ad92f52a |
| .PhaseName | Job |
| .PipelineStartTime | 2019-04-01 20:03:55+00:00 |
| .PlanID | 5e42ea59-b1aa-4240-88cf-a443c0ac38d7 |
| .PullRequest.IsFork | False |
| .PullRequest.PullRequestId | 17 | _note: only for Azure Repos?_
| .PullRequest.PullRequestNumber | | _note: only for GitHub?_
| .PullRequest.SourceBranch | refs/heads/users/raisa/new-feature |
| .PullRequest.SourceRepositoryURI | https://dev.azure.com/ouraccount/_git/OurProject |
| .PullRequest.TargetBranch | refs/heads/master |
| .ServerType | Hosted |
| .TaskDefinitionsURI | https://dev.azure.com/mattc-demo/ |
| .TaskDisplayName | Bash |
| .TaskInstanceId | efa2bfe1-554a-50c8-79b6-ef106ad3c7c2 |
| .TaskInstanceName | `Bash`, `04bff6ce5394c41b0c048826688170a` |
| .TeamFoundationCollectionURI | https://dev.azure.com/mattc-demo/ |
| .TeamFoundationServerURI | https://dev.azure.com/mattc-demo/ |
| .TeamProject | Test1 |
| .TeamProjectID | 0807fc91-4393-482d-9e23-defdbb7d0857 |
| .TimelineID | 5e42ea59-b1aa-4240-88cf-a443c0ac38d7 |
| .TotalJobsInPhase | 1 |
| .WorkFolder | D:\a |

#### Misc
| Variable | Example data | Keep, cut, or rename | Notes |
|----------|--------------|----------------------|-------|
| Common.TestResultsDirectory | D:\a\1\TestResults |
| Endpoint.URL.SystemVSSConnection | https://dev.azure.com/mattc-demo/ |
| RequestedForID | 58d417da-d63e-4df5-9884-72fd1e85590b | _note: only set for RM_ |
| Task.DisplayName | Bash |
| TF_BUILD | True |

### New variables

| New variable | Description | Special notes |
|--------------|-------------|---------------|
| CI | Set to "true" to match industry expectation for CI systems | Environment only, not available in expressions
| Pipeline.Provider | Set to "Azure" to differentiate from other CI systems
| Pipeline.Workspace | Root directory where all source, artifacts, etc will be placed
| Pipeline.Run.Url | https:// URL to pipeline run | [requested](https://twitter.com/_a__w_/status/1102802095474827264)
| Pipeline.Url | https:// URL to pipeline definition
| Pipeline.JobDisplayName | Matches what's in the UI
| ... | Commit hash of target branch
| ... | Merge commit
| _TODO_

### New namespace for variables

We'll introduce a new Pipeline.\* namespace for variables.
We'll bring forward the parts of Build.\* and Release.\* that make sense.
*TODO: define what those are.*
This is the desired future state: all relevant variables are available under Pipeline.\*, Agent.\*, or System.\*.
Some of these may be set conditionally, such as "only for pipelines triggered by a PR".

### Task migration

We'll audit in-box tasks for dependencies on "am I running in Build or Release?"
This behavior must be removed and replaced with correct behavior for a unified pipeline.
At least one (the VS Test task) depends on the System Host Type.
We'll introduce a new System Host Type, "Pipeline", which tasks can use to conditionally switch to new behavior.
*TODO: validate that this is a sane strategy. Right now it's a proposal.*

We'll introduce a task.json construct to signify "Pipeline-aware".
Once there are no remaining dependencies on deprecated variables, tasks should declare they are Pipeline-aware.
In-box tasks will be required to do so by some date.
We'll run an outreach campaign to push Marketplace tasks to do the same.

### Pipeline migration

We'll introduce a "compat mode" flag on the Pipeline level.
All existing designer pipelines and all YAML pipelines will default to "compat mode".
When a pipeline runs in compat mode, both the old and new namespace variables are injected.
This way, older tasks and scripts are not broken, but the new world is available.
In non-compat mode, only the Pipeline.\* variables are injected -- not Build.\* or Release.\*.

If any task in a pipeline is not Pipeline-aware, the flag cannot be unset on the pipeline.
Once the pipeline is free of non-Pipeline-aware tasks, it becomes a user option to change.
Eventually, we'll run an outreach campaign to instruct users to update their pipelines to turn off compat mode.
This may include injecting warnings in the pipeline run.

When a pipeline has compat mode turned off, non-Pipeline-aware tasks cannot be added.
In the designer, we give immediate feedback, and for YAML, we throw a YAML-compile-time failure.

For the designer, the compat mode flag is a UI checkbox.
Newly created pipelines will default to having compat mode off.

For YAML, it's a `version: 2` keyword at the root level of the file.
YAML v2 may also introduce other breaking changes; those are [documented elsewhere](https://github.com/Microsoft/azure-pipelines-yaml/pull/92).

## Variable scope

We'll ensure everything under System.\* is available at orchestration time, including for pipeline run numbers.
Variables under Pipeline.\* will also be available at orchestration time including run numbers.
*TODO: Ensure this is possible.*
Agent.\* variables are never available at orchestration time.
