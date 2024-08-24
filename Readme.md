# Deploying a .NET 8 Application to Linux Azure App Service with ARM Templates: A Step-by-Step Guide

This is a series post. We are covering these topis around ARM template deployment for .NET 8 applications:

* Deploy to Azure App Service - this post, keep reading
* [Deploy to a slot of the Azure App Service](./DeployToSlot.md)
* [Swap the slots](./SwapSlot.md)

I’ve been exploring a straightforward method to deploy a `.NET 8` application to a Linux Azure App Service using an ARM template. My goal was to streamline the deployment process by avoiding the use of the [az webapp CLI](https://learn.microsoft.com/en-us/cli/azure/webapp/deployment?view=azure-cli-latest), which involves interactive steps. By using ARM templates, I aimed for a fully automated solution, perfect for remote environments like build pipelines.

Initially, I thought it would be easy to find guidance on this topic and expected to find a comprehensive post. To my disappointment, I found little information available. Driven by the need for clarity and simplicity, I decided to write this post to fill the gap and provide a clear example for others in the same situation.

## Prerequisite

1. Prepare the Application:

    1. **Use the source code in [src/MyAPI](./src/MyAPI/) for a .NET 8 application.** It outputs `Hello world` in the browser.

    1. **Build and Publish locally:**

        ```shell
        dotnet publish -c Release
        ```

    1. **Archive the Published Files:**

        ```shell
        Compress-Archive src\MyAPI\bin\Release\net8.0\publish\* out\v1.zip
        ```

1. Make it public available

    Since the ARM template requires access to remotely hosted packages, I uploaded it to a public Azure Blob Storage to allow anonymous access:

    ```url
    https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v1.zip
    ```

    Note: You can protect the artifacts with a SAS token, but this is beyond the scope of this post.

Okay, we are ready.

## Build the ARM template

1. App Service Plan

    To deploy an app service in Azure, you first need an App Service Plan. Here’s a sample configuration:

    ```jsonc
    {
        "type": "Microsoft.Web/serverfarms",
        "apiVersion": "2021-02-01",
        "name": "[variables('servicePlanName')]",
        "location": "[variables('location')]",
        "sku": {
            "name": "P1V3"
        },
        "kind": "linux",
        "properties": {
            "reserved": true
        }
    }
    ```

    This configuration uses two variables: name and location. The name will be referenced by the app service later, while the location should match the resource group location. For more details, see [the full template](./ARM/initial.json).
    
    The SKU P1V3 is selected to support slot swapping. The kind is set to `linux` because the plan is intended for Linux environments, and the `reserved` property must be `true` for Linux:

    > If Linux app service plan true, false otherwise.	

    For additional details, refer to the [official documentation](https://learn.microsoft.com/en-us/azure/templates/microsoft.web/serverfarms?pivots=deployment-language-arm-template#appserviceplanproperties-1) for more details.

1. App Service

    Here’s the resource configuration for the app service itself:

    ```jsonc
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2018-02-01",
      "name": "[parameters('siteName')]",
      "location": "[variables('location')]",
      "kind": "app",
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('servicePlanName'))]",
        "siteConfig":{
          "linuxFxVersion": "DOTNETCORE|8.0"
        }
      },
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('servicePlanName'))]"
      ]
    }
    ```

    - The `serverFarmId` property links the app service to the specified service plan.
    - To run a .NET 8 application, the `siteConfig` must include the `linuxFxVersion` with the value `DOTNETCORE|8.0`. This configuration ensures that the app service knows how to bootstrap the .NET 8 application.
    - It’s crucial to declare the dependency on the app service plan using the `dependsOn` property. This ensures that the app service is deployed only after the service plan is successfully created.


1. Application Deployment

    With the service plan and app service configured, you're ready to deploy the application. To do it, you can use the `onedeploy` extension. Below is an example configuration:

    ```jsonc
    {
        "type": "Extensions",
        "apiVersion": "2022-09-01",
        "name": "onedeploy",
        "properties": {
            "packageUri": "https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v1.zip",
            "type": "zip"
        },
        "dependsOn": [
            "[concat('Microsoft.Web/sites/', parameters('siteName'))]"
        ]
    }
    ```

    **Key Points:**

    - **`packageUri`:** Specifies the location of the deployment package, which is a ZIP file in this case.
    - **`type`:** Indicates the format of the deployment package (`zip`).
        - It took me a while to findout what are supported values. Here is [Kudu publish API reference](https://learn.microsoft.com/en-us/azure/app-service/deploy-zip?tabs=arm#kudu-publish-api-reference).
    - **`dependsOn`:** Ensures that the deployment happens only after the app service has been created. This is crucial to maintain the correct deployment sequence.

    By defining `dependsOn`, you establish a clear order of operations, ensuring that the app service is fully provisioned before the deployment package is applied.

Alright, putting all those together into [initial.json](./ARM/initial.json), we can deploy it. Here I am using the Azure CLI to run the template, it is possible to run it by all other means too. Also, I am running this in PowerShell, you might need to change the way variables are used accordingly:

```shell
# login first
az login
# you might want to setup your subscription
az account set -s 'Your Subscription Name'

# Define the resource group name and location
$rgName="my-dotnet8-app-demo" 
$location="West US"

# Create the resource group
az group create -n $rgName --location $location

# Deploy resource into the resource group
az deployment group create -n manual-deploy -g $rgName --template-file .\initial.json
```

## Next Steps

* [Deploying a .NET 8 Application to an Azure App Service Slot with ARM Templates: A Step-by-Step Guide](./DeployToSlot.md)
* [Swapping App Service Slots with ARM Templates: A Step-by-Step Guide](./SwapSlot.md)

⭐ Star this repo if you like the content.