# Azure Pipelines templates to help you author PowerShell modules

Let's say you're writing a PowerShell module called `HelloWorld` that you'd like to publish into your company's private PowerShellGet gallery.

You've got a copy of your source code saved into [Azure Repos](https://katiekodes.com/what-is-ado/#azure-repos) version control _(Git)_, now you'd like to author an [Azure Pipelines](https://katiekodes.com/what-is-ado/#azure-pipelines) CI/CD pipeline to "lint," "test," "build," and "publish" your module each time you update the contents of the Git repository.

## Starting codebase

Your source code's file structure might look something like this:

```
/
├─ .cicd
│  └─ my_pipeline.yml
├─ src
│  └─ my_module
│     ├─ Private
│     │  └─ Get-CatFactArray.ps1
│     ├─ Public
│     │  ├─ Get-CatFact.ps1
│     │  └─ Get-Greeting.ps1
│     ├─ Tests
│     │  └─ Public
│     │     ├─ Get-CatFact.Tests.ps1
│     │     └─ Get-Greeting.Tests.ps1
│     ├─ HelloWorld.psd1
│     └─ HelloWorld.psm1
├─ .gitignore
└─ README.md
```

## Writing a CI/CD pipeline YAML file

Here's what you might write into the YAML file that defines your Azure Pipeline _(in this case, `/.cicd/my_pipeline.yml`)_ if you wanted to automatically build and publish your module to your company's gallery every time you edit the `main` branch of your repository:

```yaml
name: "Build out the HelloWorld module"


trigger:
  # Fire this CI/CD pipeline every time there are updates to the main branch of its associated Git repository
  branches:
    include:
      - "main"


pool:
  # Run on Linux because it's cheaper than Windows
  vmImage: "ubuntu-latest"


resources:
  repositories: 
    # Handy templates from Katie Kodes
    - repository: "psCicdHelpersRepo"
      type: "github"
      name: 'kkgthb/powershell-module-cicd-templates-azure-pipelines'
      ref: 'refs/heads/main'
      endpoint: 'REPLACE_THIS_WITH_THE_NAME_OF_A_GITHUB_TYPED_SERVICE_CONNECTION_IN_YOUR_AZURE_DEVOPS_PROJECT'


steps:

  # Lint.
  - template: "/templates/lint/run_and_report_psscriptanalyzer_linting.yml@psCicdHelpersRepo"
    parameters:
      sourceCodeFolderPath: "$(Build.SourcesDirectory)/src/all_my_modules/HelloWorld"

  # Test.
  - template: "/templates/test/run_and_report_pester_tests.yml@psCicdHelpersRepo"
    parameters:
      sourceCodeFolderPath: "$(Build.SourcesDirectory)/src/all_my_modules/HelloWorld"

  # Generate a version number based on sysdate at time of this CI/CD pipeline running.
  - template: "/templates/version/suggest_version_number_from_sysdate.yml@psCicdHelpersRepo"

  # Build, using chosen version number.
  - template: "/templates/build/build_module_and_cache_as_artifact.yml@psCicdHelpersRepo"
    parameters:
      sourceCodeFolderPath: "$(Build.SourcesDirectory)/src/all_my_modules/HelloWorld"
      versionNumber: $(suggest_version_number_based_on_sysdate.suggested_version_number)

  # Publish built module to Azure Container Registry.
  - template: "/templates/publish/publish_module_to_acr.yml@psCicdHelpersRepo"
    parameters:
      acrUrl: "https://REPLACE_THIS_WITH_YOUR_ACR_URL.azurecr.io"
      ado_service_connection_name: "REPLACE_THIS_WITH_YOUR_SERVICE_CONNECTION_NAME"
```

## Pick and choose

Feel free to pick and choose as many or as few of the Azure Pipelines templates I've shared here as are useful to you.

* Warning:  please store a copy of these templates somewhere safe.  I can't guarantee I'll be on the internet forever.

## Identity and Access Management

### For publishing

The only one of the Azure Pipelines templates I've shared here that requires special security steps is the one that publishes to Azure Container Registry.

If you're using it, be sure to set up an Azure DevOps service connection that uses federated credentials to log into an Entra app registration serving as the "principal" of an Azure RBAC role assignment of type `acrPush` scoped to the Azure Container Registry resource your company is using as an internal PowerShellGet gallery.

* Note:  in a corporate setting, you might need to open a ticket to have this done.

### For consuming

Be sure to create an Azure RBAC role assignment of type `acrPull` scoped to your Azure Container Registry whose "principal" is an Entra group full of human Entra users who need to be able to install and run your `HelloWorld` module.

_(And maybe another role assignment whose "principal" is an Entra group full of nonhuman Entra identities that also need to be able to install and run your HelloWorld module.)_

* Note:  in a corporate setting, you might need to open a ticket to have this done.

## Registering your Azure Container Registry as a PowerShell gallery on a desktop computer

Anyone with `acrPull` access to the corporate gallery should be able to run this PowerShell command:

```powershell
Register-PSResourceRepository -Name 'ANY_NICKNAME_THEY_LIKE' -Uri 'https://REPLACE_THIS_WITH_YOUR_ACR_URL.azurecr.io'
```

## Installing your module from Azure Container Registry on a desktop computer

Anyone with the Azure CLI already installed onto their coroprate desktop computer _(often available in your company's Windows Software Center)_ and authenticated into your company's Entra tenant through `az login` _(likely via a single sign on popup in their web browser)_ should be able to successfully confirm that your `HelloWorld` module exists in the corporate PowerShell gallery with this PowerShell command:

```powershell
Find-PSResource -Repository 'ANY_NICKNAME_THEY_LIKE' -Name 'HelloWorld'
```

They can install `HelloWorld` onto their computer with this PowerShell command:

```powershell
Install-PSResource -Repository 'ANY_NICKNAME_THEY_LIKE' -Name 'HelloWorld' -Scope 'CurrentUser'
```

* Note:  If at any point they get a "Response returned status code: Unauthorized" error message, they should make sure their Azure CLI is actually logged into the correct Entra tenant with `az account show`, `az account list`, `az account set --name 'THEIR_RELEVANT_AZURE_SUBSCRIPTION_NAME'`, etc.

## Running your function

Now when they run this PowerShell function:

```powershell
Get-Greeting
```

They might see something like this output, if that's how you coded `Get-Greeting`:

```
Hello, World.
```


## Links

- [PowerShell modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules) _(Microsoft Learn)_
- [PowerShellGet galleries](https://learn.microsoft.com/en-us/powershell/gallery/how-to/working-with-local-psrepositories) _(Microsoft Learn)_
- [Use Azure Container Registry _("ACR")_ repositories with `PSResourceGet`](https://learn.microsoft.com/en-us/powershell/gallery/powershellget/how-to/use-acr-repository)
- [Azure RBAC role assignments](https://katiekodes.com/azure-rbac-role-assignment/) _(Katie Kodes, my blog)_
- [Azure Container Registry "acrPush" Azure RBAC role](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-roles) _(Microsoft Learn)_
- [Entra service principals / app registrations](https://katiekodes.com/entra-app-registration/) _(Katie Kodes, my blog)_
- [Entra federated credentials](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) _(Microsoft Learn)_
- [Azure Pipelines service connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops) _(Microsoft Learn)_
- [Peer review of my first PowerShell function](https://katiekodes.com/powershell-code-review-first-function/) _(Katie Kodes, my blog)_