# Pipeline Artifacts YAML shortcut

**Status: Ready for dev design**

Pipeline Artifacts are the new way to move files between jobs and stages in your pipeline. They are hosted in Azure Artifacts and will eventually entirely replace FCS "Build Artifacts". Because moving files between jobs and stages is a crucial part of most CI/CD workflows, and because Pipeline Artifacts are expected to be the default way to do so, this spec proposes a YAML shortcut syntax for uploading and downloading artifacts.

In this document, `artifact` refers specifically to uploading Pipeline Artifacts from the current pipeline. `download` refers to downloading Pipeline Artifact artifacts from the current pipeline and from other Azure Pipeline. Other pipeline systems (e.g. Jenkins) will be handled as [pipeline resources](pipeline-resources.md).

Artifacts are distinct from other `resources` types, including `containers`, `repositories`, `packages`, and `feeds`.

## Uploading artifacts: `upload`

`upload` is a shortcut for the [Upload Pipeline Artifact](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact.md) task (formerly called "Publish Pipeline Artifacts"). It will upload files from the current job to be used in subsequent jobs or in other stages or Pipeline.

### Schema

```yaml
- upload: string # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to upload
  artifact: string # identifier for this artifact (no spaces allowed), defaults to the job name
  prependDirectory: string # a directory path that will be prepended to all uploaded files
  seal: boolean # if true, finalizes the artifact so no more files can be added after this step
```

### Example

```yaml
- upload:
  - **/bin/*
  - **/obj/*
  artifact: webapp
  seal: true
```

---

## Downloading artifacts: `download`

`download` is a shortcut for the [Download Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/download-pipeline-artifact) task.

It will download artifacts uploaded from a previous job, stage, or from another pipeline.

### Artifact download location

#### YAML / unified pipelines
Artifacts are downloaded to the pipeline's workspace by default. We introduce a new variable for unified pipelines: `$(Pipeline.Workspace)`. `Pipeline.Workspace` is one level up from `$(System.DefaultWorkingDirectory)`; on hosted, it corresponds to `c:\agent\_work\1\`.

Each artifact is given its own directory e.g. `$(Pipeline.Workspace)\myartifact` for an artifact named `myartifact`. Artifacts coming from other pipelines are each given one directory per pipeline e.g. `$(Pipeline.Workspace)\some-other-pipeline\someartifact` for the `someartifact` artifact of the `some-other-pipeline` pipeline. The pipeline portion of the path comes from the ID of the pipeline's `resource` in YAML.

Example:
```yaml
resources:
  pipelines:
  - pipeline: mypipe

jobs:
- job: makeartifact
  steps:
  - script: ./build.sh
  - upload: outputs/**/*

- job: use1artifact
  dependsOn: makeartifact
  steps:
  # by default, a download step is injected -- see later section
  - script: ls $(Pipeline.Workspace)
    # this listing shows a folder called `makeartifact`, since the prior job didn't specify a name

- job: use2artifacts
  dependsOn: makeartifact
  steps:
  - download: mypipe   # downloads all artifacts from `mypipe`
  - download: current  # must include this, since by including a download step, we don't get automatic behavior anymore
  - script: ls $(Pipeline.Workspace)
    # this listing shows two folders: `makeartifact` and `mypipe`
```

#### Controlling output path

If the `path` key is specified, it's a relative path from `$(Pipeline.Workspace)`. Directory names are not automatically injected by the pipeline anymore (but of course, directories present in the artifact itself are still used).

Example:
```yaml
jobs:
- job: makeartifact
  steps:
  - script: ./build.sh
  - upload: outputs/**/*

- job: useartifact
  dependsOn: makeartifact
  steps:
  - download: current
    path: foo
  - script: ls $(Pipeline.Workspace)
    # listing shows one folder, "foo"
```

#### Build and RM classic pipelines
No change to current behavior. Artifacts are downloaded to `$(System.DefaultWorkingDirectory)`, which is the sources folder on Build and the artifacts folder on RM.

### Schema

```yaml
- download: string # identifier for the pipeline resource from which to download artifacts, optional; "current" means the current pipeline, blank means all available pipelines (including current)
  artifact: string # identifier for the artifact to download; optional
  patterns: string # a minimatch path or list of [minimatch paths](tasks/file-matching-patterns.md) to download; if blank, the entire artifact is downloaded
  path: string # the directory in which to download files, defaults to $(Pipeline.Worksapce); if a relative path is provided, it will be rooted from $(Pipeline.Workspace)
  displayName: string # friendly name displayed in the UI
  name: string # identifier for this step (A-Z, a-z, 0-9, and underscore)
  condition: string
  continueOnError: boolean # 'true' if future steps should run even if this step fails; default to 'false'
  enabled: true # whether or not to run this ste; defaults to 'true'
  timeoutInMinutes: number
  env: { string: string } # list of environment varibles to add
```

### Examples

```yaml
- download: current
  artifact: webapp
- download: tools-pipeline
```

## More details about Pipeline Artifacts

### Default and named artifacts

If no name is specified, the uploaded artifact should be named the same as the job name. You can also create multiple artifacts, each with their own name.

All artifacts (including the default) are automatically downloaded to each subsequent job's workspace directory (`$(Pipeline.Workspace)`). You can limit the artifacts downloaded for any job by adding a `download` step to the beginning of the job. Once you use `download` to override and download a specific artifact, all automatic artifact download behavior is disabled and you need to specify any and all artifacts you intend to download in the job.

Or to avoid downloading any of the artifacts at all:

```yaml
- download: none
```

### Multi-upload artifacts

You can add files to any artifact multiple times in the same job, in different jobs in the same stage, and in different stages in the same pipeline, or you can `seal` an artifact at any time to prevent further files from being added.

## Examples

### Upload a build artifact and use it in a deployment

This is a simple pipeline that include a build job and a deployment job. The built artifact is provided to the deployment job using the default Pipeline Artifact.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/*
- job: Deploy
  steps:
  - script: |
      ./my-deploy-script.sh $(Pipeline.Workspace)/Build/
```

### Specify a custom location for a build artifact

You can control the location where artifacts are downloaded using the `path` key.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/*
- job: Deploy
  steps:
  - download:
    path: from-build/
  - script: |
      ./my-deploy-script.sh $(Pipeline.Workspace)/from-build/
```

### Add to an artifact multiple times

You can add files to both the default artifact and to named artifacts until the end of the pipeline or until an `upload` step is run with the `seal` key set to `true`.

```yaml
- job: buildCore
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/*
    prependPath: netcore/
- job: buildNetFx
  steps:
    - task: VSBuild@1
      inputs:
        solution: MySolution.sln
    - upload: bin/*
      prependPath: netfx/
      seal: true
- job: Deploy .NET Core
  dependsOn: buildCore
  steps:
  - script: |
      ./my-deploy-script.sh $(Pipeline.Workspace)/buildCore/netcore/
```

### Upload a named artifact

You can give an artifact a name, and you can upload multiple named artifacts. All artifacts are downloaded unless you specify a `download` step to limit the artifacts that are downloaded.

```yaml
- job: Build
  steps:
  - script: dotnet publish --configuration $(buildConfiguration)
  - upload: bin/WebApp/*
    artifact: WebApp
  - upload: bin/MobileApp/*
    artifact: MobileApp
- job: Deploy
  steps:
  - download:
    name: WebApp
  - download:
    name: MobileApp
  - script: ./my-deploy-script.sh $(Pipeline.Workspace)/WebApp/
  - script: ./my-xamarin-script.sh $(Pipeline.Workspace)/MobileApp/
```
