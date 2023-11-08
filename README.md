# APIM Developer Portal Migration

## Steps for Azure DevOps

1. Create [Service Connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml#create-a-service-connection) in Azure DevOps for each of the APIM instances you want to migrate from and to.

2. Create a [variable group](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=classic#create-a-variable-group) in Azure DevOps called ***apim-dev-portal*** with the following variables:

    | Variable Name | Description |
    | ------------- | ----------- |
    | APIM_INSTANCE_NAME | The name of the APIM instance to migrate from |
    |REPOSITORY_NAME|The name of the Azure DevOps repository to save the artifacts to|
    |RESOURCE_GROUP_NAME|The name of the resource group the APIM instance is in|
    |SERVICE_CONNECTION_NAME|The name of the Azure DevOps service connection to use to connect to the APIM instance|
    |TARGET_BRANCH|The name of the branch to raise the PR against|

3. Create a variable group in Azure DevOps in the format ***apim-dev-portal-{env}*** for each of the environment you want to migrate to with the following variables:

    | Variable Name | Description |
    | ------------- | ----------- |
    | APIM_INSTANCE_NAME | The name of the APIM instance to migrate to |
    |RESOURCE_GROUP_NAME|The name of the resource group the APIM instance is in|
    |SERVICE_CONNECTION_NAME|The name of the Azure DevOps service connection to use to connect to the APIM instance|

4. Create an [environment](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops#create-an-environment) in Azure DevOps for each of the environments you want to migrate to. Add required [gates](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops&tabs=check-pass#approvals) to the environment.

5. Create a pipeline in Azure DevOps with the yaml from file ***capture.yaml***. This will create a pipeline that will capture the APIM developer portal snapshot from the source environment and raise a PR against the repository specified in the variable group.

6. Run the pipeline with the following parameters:

    OUTPUT_FOLDER_PATH: The name of the folder to save the snapshot to in the repository.

    *Note*: Pipeline will need approval to access the variable group for the first time.

7. Create a pipeline in Azure DevOps with the yaml from file ***release.yaml***. This will create a pipeline that will deploy the APIM developer portal snapshot to the target environment(s).

8. Link the variable groups created for each environment to the pipeline. Edit the pipeline, select Triggers from the ellipsis menu and navigate to the Variables tab. Add the variable groups to the pipeline.

9. Create url overwrite files for each environment in the format ***urls.{env}.json***. The contents of the file should be as

    ```json
    {
        uri: "https://uri1.com,https://uri2.com"
    }
    ```

    Update the existing uris in the snapshot in the file ***existingUrls.json***

    *Note*: This is an optional step and is only required if you want to overwrite the urls in the snapshot with the urls in the file.

10. Run the pipeline with the following parameters:

    OUTPUT_FOLDER_PATH: The name of the folder the snapshot was saved to in the repository.

    ENVIRONMENTS: The name of the environments to deploy to. For ex:

    ```json
        - dev
        - stage
        - prod
    ```

    These should match the suffix of the names of the variable groups created in step 3.

## Steps for GitHub Actions

1. Create Service Principals for each of the APIM instances you want to migrate from and to. The service principal will need the following permissions:

    ```azurecli
    az ad sp create-for-rbac -n "{name_of_the_sp}" --role Contributor --scopes /subscriptions/{subscriptionId}/resourceGroups/{apim_resource_group} --sdk-auth
    ```

2. Create environments for each of the apim instances under *{repository} -> Settings -> Environments* with the below secrets:

    | Secret Name | Description |
    | ------------- | ----------- |
    |APIM_INSTANCE_NAME |The name of the APIM instance to migrate from |
    |RESOURCE_GROUP_NAME|The name of the resource group the APIM instance is in|
    |AZURE_CLIENT_ID|The client id of the service principal|
    |AZURE_CLIENT_SECRET|The client secret of the service principal|
    |AZURE_SUBSCRIPTION_ID|The subscription id of the apim resource |
    |AZURE_TENANT_ID|The tenant id of the service principal|

    *Note:* The names of the environments can be dev, stage etc. If using different names, update the capture.yaml and release.yaml for the environment names. This would also be a good time to setup [deployment protection rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#deployment-protection-rules) if you wish in the environment settings of GitHub.

3. Grant permissions for the actions to create a PR. Set *Read and write permissions* and "Allow GitHub Actions to create and approve pull requests" under *{repository} -> Settings -> Actions -> General -> Workflow permissions*.

4. Update the *release.yaml* to reflect the stages you want to deploy to. 

5. By default the folder used to store the Developer Portal artifacts in this repo is `artifacts` as referenced in the [release-with-env.yaml file](.github/workflows/release-with-env.yaml#L22). If you would like to use a different folder name, you will need to update that file first.

6. Create url overwrite files for each environment in the format ***urls.{env}.json***. The contents of the file should be as

    ```json
    {
        uri: "https://uri1.com,https://uri2.com"
    }
    ```

    Update the existing uris in the snapshot in the file ***existingUrls.json***

    *Note*: This is an optional step and is only required if you want to overwrite the urls in the snapshot with the urls in the file.

7. Run the `capture.yaml` pipeline and provide a folder to store the artifacts in, the default is `artifacts`. That will pull in the Developer Portal artifacts that are in the current dev environment (this is set in the [capture.yaml](.github/workflows/capture.yaml#L15) file, so if you want to pull from a different environment update that). Once that is complete, you can see the PR that was created so you can merge it. Once that is merged the `release.yaml` pipeline will automatically trigger.
