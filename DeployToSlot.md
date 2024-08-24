# Deploying a .NET 8 Application to an Azure App Service Slot with ARM Templates: A Step-by-Step Guide

In my previous post, we covered deploying a .NET 8 application to an Azure App Service using ARM templates. If you missed it, you can read it [here](./Readme.md). Today, we’ll take the next step by deploying an application to a specific slot within the App Service.

Deploying to a slot allows you to test new versions of your app in an isolated environment before moving them into production. In the upcoming post, we'll explore how to swap this slot with the production environment using ARM templates.

If you prefer to take a look at the template on your own, refer to [staging-slot.json](./ARM/staging-slot.json)

## Prerequisites

Ensure you have an existing App Service deployed. We will be deploying a new version of the application to a slot, such as a staging slot, as illustrated below:

![slot](./imgs/AfterSwap.png)

## Steps

1. Prepare the Application

    Refer to [Prepare the Application](./Readme.md#prerequisite) to build and upload the application. Edit the source code to output text `Hello World! v2` for example, and get the package of `v2.zip` uploaded to the blob storage:

    ```
    https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v2.zip
    ```

1. Create an ARM template to deploy an empty stage:

    ```jsonc
    {
      "type": "Microsoft.Web/sites/slots",
      "apiVersion": "2018-02-01",
      "name": "[concat(parameters('siteName'), '/', parameters('slotName'))]",
      "location": "[variables('location')]",
      "properties": {
      }
    }
    ```

    Notice, the name of the slot in form of `siteName/slotName`, and make sure the site name is correct.

1. Append package to deploy, by using `onedeploy` extension again:

    ```jsonc
    {
        "type": "Extensions",
        "apiVersion": "2022-09-01",
        "name": "onedeploy",
        "properties": {
        "packageUri": "https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v2.zip",
            "type": "zip"
        },
        "dependsOn": [
            "[concat('Microsoft.Web/sites/', parameters('siteName'),'/slots/', parameters('slotName'))]"
        ]
    }
    ```

    The key here is the form up the correct dependency declare.

1. Deploy the ARM template for testing

    Once ready, let's deploy the template:

    ```powershell
    # login first
    az login
    # you might want to setup your subscription
    az account set -s 'Your Subscription Name'

    # Define the resource group name, making sure it is correct
    $rgName="my-dotnet8-app-demo" 

    # Start the deployment
    az deployment group create -n manual-deploy -g $rgName --template-file .\staging-slot.json
    ```


