# lcaf-component-provider_az-pipeline_azdo

## Use reusable templates in Azure DevOps pipelines

To use the reusable templates from this repository from a pipeline in another repository

```yaml
# At the top of your pipeline, define this resource of type repositories
resources:
  repositories:
    # This is the repository where ADO pipeline templates and common bash functions are stored
    - repository: utils
      type: github
      name: 'launchbynttdata/lcaf-component-provider_az-pipeline_azdo'
      ref: feature/1456
      endpoint: 'launchbynttdata-github-token'

# Then use the templates in the pipelines as below
stages:
  # Use absolute path
  - template: /iac/stages/terragrunt.yaml@utils
    parameters:
      deployEnvironments:
        - environment: sandbox
          region: eastus
          environmentNumber: "000"
          azureEnvironment: sandbox-env
          serviceConnection: service-connection
```

For referring to other templates inside a template, you can use relative path. For example consider the stage template [terragrunt.yaml](iac/stages/terragrunt.yaml) which refers to the step template [terragrunt-plan.yaml](iac/steps/terragrunt-plan.yaml). The relative path to refer to the step template from the stage template is as below

```yaml
- template: ../steps/terragrunt-plan.yaml
  parameters:
      serviceConnection: ${{ deployEnv.serviceConnection }}
      environment: ${{ deployEnv.environment }}
      region: ${{ deployEnv.region }}
      environmentNumber: ${{ deployEnv.environmentNumber }}
      workingDirectory: "$(Pipeline.Workspace)/main"
```

## Variables
Variables can be defined at different levels in Azure DevOps pipelines.

### Global Variables
Variables defined at the top of the pipeline are available to all stages and jobs in the pipeline. Remember that these
variables cannot be overridden in the Azure Devops UI by creating variables with the same name. They can only be changed in the pipeline yaml.

### Variables in ADO UI
You can create variables while editing the pipeline in Azure Devops UI and adding new variables. Those variables can be overridden during each pipeline run.

Remember if the same variable is defined at the top of the pipeline yaml, then the variable defined in the UI will not take effect

### Pre defined variables
This is a list of all pre defined variables provided by Azure [predefined variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml)

### Set variables in tasks

By default variable set in one task can be used only by tasks in the same job.
Set variables
```
- bash: |
    echo "##vso[task.setvariable variable=myVar;]foo"
```

Get variables

```
- bash: |
    echo "You can use macro syntax for variables: $(myVar)"
```

Variables can also be set to be used in future jobs and stages. See more details at [User Defined variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-variables-scripts?view=azure-devops&tabs=bash)

## Runtime Parameters

### Parameters in pipelines

Parameters can be defined at the top of the pipeline and can be used in the pipeline. These parameters are shown in the Azure DevOps UI when the pipeline is run as text boxes and radio buttons etc. and can be overridden.

User can pass in the `pool` parameter while manually triggerring the pipeline from the UI

```yaml
parameters:
  - name: pool
    type: string
    default: 'container-pool'
```

### Parameters in templates

Templates can accept inputs in form of parameters. There are various data types of parameters supported by Azure. More details can be found at [parameter types](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/template-parameters?view=azure-devops#parameter-data-types)

## Best Practices

1. Templates can access the variables defined either at the top of the pipelines or defined in the ADO pipeline UI. But its recommended that we use parameters to pass in inputs to templates rather than variables. That makes the templates more readable and the users can be sure what parameters to pass into the templates to use them.

2. When a parameter type `object` is used in a template, user can pass any arbitrary values. To make it more readable, its recommended to provided documentation or default values

3. While checking out multiple repositories in a pipeline, set the `workingDirectory` for the tasks in the reusable templates to `$(Pipeline.Workspace)/main` where `main` is the `self` repository. Also checkout the repos like below
  
  ```yaml 
  - checkout: self
    path: main
  - checkout: utils
    path: utils
  ```