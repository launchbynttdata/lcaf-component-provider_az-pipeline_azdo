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
- template: /applications/steps/terragrunt-plan.yaml
  parameters:
      serviceConnection: ${{ deployEnv.serviceConnection }}
      environment: ${{ deployEnv.environment }}
      region: ${{ deployEnv.region }}
      environmentNumber: ${{ deployEnv.environmentNumber }}
      workingDirectory: "$(Pipeline.Workspace)/main"
```

## Variables
Variables can be defined at different levels in Azure DevOps pipelines - root, stage or job level.

Different syntax to define variables
- macro syntax `$(var)`: gets processed during `runtime` before a task runs. Runtime happens after template expansion.
  - if `$(var)` can't be replaced, `$(var)` won't be replaced by anything.
  - macro syntax can only be used for steps, jobs and stages. Cant be used for resources or triggers.
  - Macro variables are only expanded when they're used for a value, not as a keyword. Values appear on the right side of a pipeline definition. The following is valid: `key: $(value)`. The following isn't valid: `$(key): value`.
- runtime expression `$[variables.var]` : gets processed during runtime
  - Intended to run with conditions or expressions
  - Runtime expression variables silently coalesce to empty strings when a replacement value isn't found.
  - Runtime expression variables are only expanded when they're used for a value, not as a keyword. Values appear on the right side of a pipeline definition. The following is valid: `key: $[variables.value]`. The following isn't valid: `$[variables.key]: value`.
  - The runtime expression must take up the entire right side of a key-value pair. For example, `key: $[variables.value]` is valid but `key: $[variables.value] foo` isn't.
- template expression `${{ variables.var }}` gets processed at compile time before runtime starts
  - Template variables silently coalesce to empty strings when a replacement value isn't found
  - Template expressions, unlike macro and runtime expressions, can appear as either keys (left side) or values (right side). The following is valid: `${{ variables.key }} : ${{ variables.value }}`

**Note:** When you define the same variable in multiple places with the same name, the most locally scoped variable wins. So, a variable defined at the job level can override a variable set at the stage level. A variable defined at the stage level overrides a variable set at the pipeline root level. A variable set in the pipeline root level overrides a variable set in the Pipeline settings UI.

### Global Variables

Variables defined at the top of the pipeline are available to all stages and jobs in the pipeline. Remember that these
variables cannot be overridden in the `Azure Devops UI `by creating variables with the same name. They can only be changed in the pipeline yaml.

### Variables in ADO UI
You can create variables while editing the pipeline in Azure Devops UI and adding new variables. Those variables can be overridden during each pipeline run.

Remember if the same variable is defined at the top of the pipeline yaml, then the variable defined in the UI will not take effect

### Pre defined variables
This is a list of all pre defined variables provided by Azure [predefined variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml)

**Note**: There are a few pre-defined variables that are only available to tasks if defined directly in the pipeline yaml, however not available to the tasks defined in the templates.

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

## Secrets
One can access the secrets stored in the Azure KeyVault directly in the pipeline steps. Users can use below task to retrieve secrets from the KeyVault

```yaml
#Secrets will be availabe as variables to the subsequent steps in the job
- task: AzureKeyVault@1
  inputs:
    # Service Connection (Service Principal) must have required access to the KeyVault
    azureSubscription: 'repo-kv-demo'                    ## YOUR_SERVICE_CONNECTION_NAME
    KeyVaultName: 'kv-demo-repo'                         ## YOUR_KEY_VAULT_NAME
    # Accepts comma seperated list of secret names
    SecretsFilter: 'secretDemo'                          ## YOUR_SECRET_NAME. Default value: *
    # Runs the task before the job execution begins. Exposes secrets to all tasks in the job, not just tasks that follow this one.
    RunAsPreJob: false
```

Use the secrets in the subsequent steps as below

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: 'run'
    projects: '**/*.csproj'
  env:
    mySecret: $(secretDemo)

- bash: |
    echo "Secret Found! $MY_MAPPED_ENV_VAR"
  env:
    MY_MAPPED_ENV_VAR: $(mySecret)
```



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
