# Step Target

Steps run either on the host that is running the agent.worker or if the step is part of a [Container Job](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/container-phases?view=azure-devops&tabs=yaml) it runs inside of the container specific on the job.  Steps are not able to target the host if running in a container job or other containers if using [services](./sidecar-containers.md).

The goal of a step target is to give the pipeline author flexibility in where a given step in the job runs.  This not only proivdes additional flexibility for pipeline authors, it can also be used by template authors to drive certian security and compliance scenarios in their pipeline.

## Scenarios

1. I have a service defined on my job and I want to run a script that targets the container that service is running in.
2. I have authored a template and I want all steps defined by the consumer of my template to run inside a container after I have run a step on the host to disable the network for that container in order to make sure their build does not pull down any extra dependencies.

## YAML Syntax

**example:** a container job using [services](./sidecar-containers.md) with one step targeting a service container.

```yaml
jobs:
- job: MyJob
  pool:
  container: 
    image: python:latest
    name: python-builder

## Multi-container support here
  services:
    redis:
      image: redis:alpine
      ports:
      - "6379"
    postgres:
      image: postgres:9.4
      volumes:
      - db-data:/var/lib/postgresql/data
      env:
        FOO: bar
## End multi-container

  steps:
  - script: ...
  	displayName: Configure postrgres
  	target: postgres
  - script: ...
    displayName: Run tests
  - script: ...
  	target: host # reserved name could also be agent
  
```

**example:** A template that runs steps from a consumer in a contianer with no network

```yaml
## job-template.yml
parameters:
# Job schema parameters - https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=vsts&tabs=schema#job
  cancelTimeoutInMinutes: ''
  condition: ''
  continueOnError: false
  container: ''
  dependsOn: ''
  displayName: ''
  steps: []
  pool: ''
  strategy: ''
  timeoutInMinutes: ''
  variables: []
  workspace: ''
  
- jobs
  job: ${{ parameters.name }}
  # common jobs parameters
  steps:
  # remove the docker network from the job container need an easy way to determine the job network and job container
  - script: docker network disconnect <job network> <job container>
    target: host # reserved name for the host the worker is running on
  
  # copy each proerty from each step skipping the target property
  - ${{ each step in parameters.steps }}:
    - task: ${{ step.task }} 
  	  displayName: ${{ step.displayName }}
      name: ${{ step.name }}
      condition: ${{ step.condition }}
      continueOnError: ${{ step.continueOnError}}
      enabled: ${{ step.enabled }}
      timeoutInMinutes: ${{ step.timeoutInMinutes }}
      inputs: ${{ step.inputs }}
      env: ${{ step.env }}

## jobs-template.yml
parameters:
  jobs: []

jobs:
- ${{ each job in parameters.jobs }}:
  - template: ./job-template.yml
    parameters: 
      # pass along parameters
      ${{ each parameter in parameters }}:
        ${{ if ne(parameter.key, 'jobs') }}:
          ${{ parameter.key }}: ${{ parameter.value }}

      # pass along job properties
      ${{ each property in job }}:
        ${{ if ne(property.key, 'job') }}:
          ${{ property.key }}: ${{ property.value }}

      name: ${{ job.job }}

## azure-pipelines.yml

resources:
  containers:
  - container: mycontianer
    image: org/mycontainer:stable

jobs:
- template: /templates/jobs-template.yml
  parameters:
  jobs:
  - job: MyBuild
  	container: mycontainer
  	steps:
  	- script: ...
  	- script: ...
```



## Notes

Need to figure out the reserved target for the agent or host

Need a standard set of pipeline variables that can be used to get the name of the job network and the job container 

Service containers probably need to map the `workspace` in the same way the job container does