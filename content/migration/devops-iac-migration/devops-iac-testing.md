# Milestone: Testing Environment for Migration Wave

#### [prev](./devops-iac-redeployment.md) | [home](./readme.md)  | [next](./devops-iac-migration.md)

## Overview
This section outlines the steps needed in order to execute an Azure DevOps Pipeline with the appropriate tasks needed for setting up a test environment for migration. This implementation focuses on rehosting a subset of the defined migration wave in order to test the server functionality with the parameters from the Azure Migrate Assessment.

The sample pipeline takes the output of an Azure Migrate Assessment and creates an isolated test environment with new VMs that are based on the configuration from the exported assessment output. The VMs that will be created via this sample pipeline are empty Marketplace images, ready for configuration and app installation.

![Migration Workflow](./src/workflow-devops-iac-milestone2.png)

## 1 Pre-Requisites
To get started, the assumption is the following:
* The [discovery](https://github.com/Azure/FTALive-Sessions/blob/main/content/migration/server-migration/scan.md) is completed for the scoped VMs.
* The [assessment](https://github.com/Azure/FTALive-Sessions/blob/main/content/migration/server-migration/assess.md) for the environment is created within Azure Migrate.
    * The assessment is the source for where the VMs in the pipeline are created. Please ensure that only the VMs that are scoped for the test environment are in the assessment. Please manually omit VMs not needed in the deployment.
* The assessment was exported as an Excel file to your local machine.
* An [Azure DevOps Organization](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/organization-management?view=azure-devops) is created and linked to your subscription in Azure.
* The [src folder](./src/) is cloned on your local machine.
* [PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.2) and Excel are installed on your local machine.


### 1.1\. Pre-Migration Tasks 
Please refer to the [Pre-Migration and Post-Migration Activities](https://github.com/Azure/FTALive-Sessions/blob/main/content/migration/server-migration/testing.md#1-pre--post--migration-activities-defined) page that should be defined before executing the test pipeline. 

General best practices when preparing for test migration in the outlined scenarios below include:

**Performing a Test Migration on an Isolated VNet**
- Define parameters needed for an isolated VNet implementation (i.e. CIDR block, NSG Ports that will open on the test subnet, etc.).
- Test the connectivity in the isolated VNet ([Guidance for identifying target VNet](https://github.com/Azure/FTALive-Sessions/blob/main/content/migration/server-migration/testing.md#23-identify-target-vnets-tests-and-migration-workflow))
- Ensure that appropriate stakeholders are given the least privilege permissions to execute the pipelines.
- Define Test Migration approach through waves of execution:
    - Understand dependencies to create migration waves/groups.
    - Define test cases.
    - Ensure Rollback plan is in place for the re-hosted VMs.
    - Make sure that test data is consistent with the data used in production.
- Clean up test resources that were deployed in an isolated VNet.

**Performing a Test Migration in a Production VNet**
- Define parameters needed for the production VNet
- Plan a maintenance window to shut down on-prem workload for test migration, set up a VNet with the parameters needed for production workloads to move to Azure
- Define test cases for the environment
- Choose non-prod VM Migration group to start the test functionality with.
- Perform tests on a smaller VM waves of the workloads first.
- Ensure Rollback plan is in place for the re-hosted VMs.


### 1.2\. Create the appropriate scripts that correlate to the [types of tests](https://github.com/Azure/FTALive-Sessions/blob/main/content/migration/server-migration/testing.md#2-migration-plan-definition) needed for your deployment:
- Smoke Test
- UAT
- Failover

![Concept Diagram](../server-migration/png/migration-workflow.PNG)

## 2 Pipeline Execution for Testing

This sample testing pipeline can be customized to fit the needs of your organization and can be modified to incorporate specific tests needed for your app validation in the Testing phase of migration.

### 2.1 PowerShell Implementation

The steps below outline the process for redeploying Azure assets through PowerShell, using the csvs as a source for the IaaS parameters.

#### 2.1.1\. Input parameters your environment using the [testing-variables.yml](./src/test-migration/testing-variables.yml) as a template.

#### 2.1.2\. Create a `testing-pipeline.yml` for resource execution using the provided [template](./src/test-migration/testing-pipeline.yml) as a baseline. Below are a description of the tasks:
Pipeline Tasks:
- Start Test Migration
    - Create isolated VNet (optional)
    - Within Powershell script:
        - Create the Migration Group Compute needed
- Validate Test Environment (manually or using test scripts)
    - Run Testing Scripts (if applicable)
        - Smoke Test
        - UAT
        - Failover
- Clean up Test Resources

### 2.2 Bicep Implementation

The steps below outline the process for redeploying Azure assets through Bicep. This implementation uses the PowerShell script implementation (2.1) as a baseline for parsing the parameters and then executes the resources based on the parameters outlined by the CSVs in order to create Azure resources using Bicep.

#### 2.2.1\. Input variables for your environment in Azure DevOps under `Pipelines` > `Library`. There you will see a variable group called `bicepPipelineVariablesDev` where you can input the appropriate parameters.
* *Note:* Make sure to update the parameters with your own environment names. 
* More info on variable groups [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml).

#### 2.2.2\. Create a `testing-pipeline.yml` for resource execution using the provided [template](./src/test-migration/bicep-test-migration-pipeline.yml) as a baseline. Below are a description of the tasks:
Pipeline Tasks:
- Create isolated VNet with Bicep deployment (optional)
- Powershell script run to parse the parameters and call Bicep templates for execution.
- Clean up Test Resources

### 2.3\. Validate Target VNet Tests
* If execute the isolated VNet Testing Pipeline, validate the pipeline has ran through the necessary tasks above.
* If utilizing the final Migration VNet, set the maintenance window and prepare migration waves for execution using the sample pipeline [template](./src/prod-migration/bicep-migration-pipeline.yml) for migration.
* If any of the tests fail within a pipeline stage, execute the Rollback plan for the migration wave.

### 2.4\. Post Test Migration Tasks 
- Validate VM migration was successful and that applications are functioning as expected
    - Perform capacity testing to ensure that functioning properly in production

### 2.5\. Expected Results 
* VNet created with specified parameters for testing its functionality before full migration of servers to Azure
* Standardized pipeline to utilize for deploying compute resources in Azure
* Pipeline provides option for executing tests within Azure on the testing resources that are deployed
* Pipeline performs clean up of test resources after validation of functionality

### 2.6\. Considerations for Customizations
The provided templates in the Azure DevOps Repo are meant to be a starter baseline for the execution of your migration waves. The defaults for the variables are based on the input CSV from Azure Migrate. Below are some additional areas for consideration if want to further customizing the templates to fit your business needs for redeploying Azure resources.

#### 2.6.1\. Modifying VM SKU and Disk Size
In the bicep templates, these options can be modified in the [Windows Bicep Template](./src/bicep/vmWindows.bicep) and [Linux Bicep Template](./src/bicep/vmLinux.bicep) in the following sections: 
* **SKU parameter:** `param sku string` can be edited (i.e. `param sku string = '2019-Datacenter'`).
* **VM Size:** `param vmSize string` can be edited (i.e. `param vmSize string = 'Standard_D2s_v3'`).
* **Disk Size:** `param osDiskSize int` can be edited (i.e. `param osDiskSize int = 127`).
* **Data Disk Size:** data disks are currently looped through in the following method based on the input CSV: 
    ```bicep
     dataDisks: [for i in range(0,length(datadisksizes)): {
        name: '${vmname}-dataDisk${i}'
        diskSizeGB: datadisksizes[i]
        lun: i
        createOption: 'Empty'
        managedDisk: {
            storageAccountType: datadisktypes[i]
        }
     }
    ```
    The disks can also be added on an individual basis in the following method:
    ```bicep
    dataDisks: [
        {
            diskSizeGB: 1023
            lun: 0
            createOption: 'Empty'
        }
    ]
    ```

#### 2.6.2\. VM Diagnostics Settings
Boot Diagnostics can be enabled or disabled in the following section:

```bicep
 diagnosticsProfile: {
    bootDiagnostics: {
        enabled: true //set as false to disable
    }
 }
```
A specific storage account can also be referenced to correlate to the VMs by adding a storage account reference: 

```bicep
var storageAccountName = 'bootdiags${uniqueString(resourceGroup().id)}'

resource bootStorage 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'Storage'
}

```
and the following under `bootDiagnostics`:
```bicep
diagnosticsProfile: {
    bootDiagnostics: {
        enabled: true //set as false to disable
        storageUri: bootStorage.properties.primaryEndpoints.blob
    }
 }
```
#### 2.6.3\. VM Passwords 
In the baseline Bicep template for the Linux and Windows machines, it currently has hard coded passwords and usernames as an example for the implementation as seen below. This path is for testing/example purposes only.


```bicep
param adminUserName string ='azureuser'
param adminPassword string ='@zureS3cur3P@ssw0rd'
```

```bicep
resource WindowsVM 'Microsoft.Compute/virtualMachines@2021-07-01' = {
    ...
  properties: {
      osProfile: {
          computerName: vmname
          adminUsername: adminUserName
          adminPassword: adminPassword
      }
    ...
  }
```

Below are the recommended paths to take when implementing:

**Storing Passwords as a Secret in Key Vault:**
1. Ensure that the Pipeline service principal has the appropriate permissions for using a key vault
    1. Permission in RBAC: `Microsoft.KeyVault/vaults/deploy/action`
1. Deploy a key vault in Azure and set up the secrets using the following guidance:
[Quickstart: Set and retrieve a secret from Azure Key Vault using Bicep](https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-bicep?tabs=CLI)
1. Pass the secret in the VM template Bicep files using the getSecret function:
    1. Ensure that your parameter for the password has the following format:
    ```bicep
    @secure()
    param adminPassword string
    ```
    1. Call the getSecret function based on the previous key vault reference:
    
    ```bicep
    resource kv 'Microsoft.KeyVault/vaults@2021-11-01-preview' = {...}
    resource secret 'Microsoft.KeyVault/vaults/secrets@2021-11-01-preview' = {...}

    resource WindowsVM 'Microsoft.Compute/virtualMachines@2021-07-01' = {
        ...
      properties: {
          osProfile: {
              computerName: vmname
              adminUsername: adminUserName
              adminPassword: kv.getSecret('<secret_name>')
          }
        ...
      }
    }
    ```
    More info on getSecret function:[Use getSecret function](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/key-vault-parameter?tabs=azure-cli#use-getsecret-function)

Using an SSH Key:
1. Generate SSH Key with `ssh-keygen` in your CLI.
1. Add the parameters below in the following format:
    ```bicep
    param authenticationType string = 'sshPublicKey'    
    @secure()
    param adminPasswordOrKey string
    ```
1. Add the `linuxConfiguration` variable to reference the SSH path
    ```bicep
    var linuxConfiguration = {
      disablePasswordAuthentication: true
      ssh: {
        publicKeys: [
          {
            path: "/home/${adminUsername}/.ssh/authorized_keys"
            keyData: adminPasswordOrKey
          }
        ]
      }
    }
    ```
1. Add the `adminPassword` and `linuxConfiguration` references to the `osProfile` for the VM: 
    ```bicep
    resource LinuxVM 'Microsoft.Compute/virtualMachines@2021-07-01' = {
        ...
      properties: {
          osProfile: {
              computerName: vmName
              adminUsername: adminUsername
              adminPassword: adminPasswordOrKey
              linuxConfiguration: ((authenticationType == 'password') ? null : linuxConfiguration)
            }
        ...
      }
    }
    ```
