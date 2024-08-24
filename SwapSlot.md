# Swapping App Service Slots with ARM Templates: A Step-by-Step Guide

In my previous posts, we discussed [how to deploy an application to an Azure App Service](./Readme.md) and [how to deploy an application to a slot](./DeployToSlot.md). In this post, we'll focus on how to swap these slots using ARM templates.

Swapping slots with ARM templates can be complex at first glance, but this guide will clarify the process. If you prefer to examine the templates directly, you can view [swap.json](./ARM/swap.json) and [swap-combo.json](./ARM/swap-combo.json). Assuming you have an app deployed with a slot, here's a high-level overview of how the ARM template works:

1. **Assign a Build Version**: Set a build version (e.g., `Ver.aaa`) to the slot.
2. **Update Production**: Set the `targetBuildVersion` property of the `Production` environment to the same build version (`Ver.aaa`).

**Why?** When you deploy a template like this, ARM follows these steps:

1. Marks the slot with the build version `Ver.aaa`.
2. Swaps the current Production environment with the slot that has the version `Ver.aaa`.

A minimal swap template would look like this:

```jsonc
"resources": [
  {
    "type": "Microsoft.Web/sites/slots",
    "apiVersion": "2018-02-01",
    "name": "[concat(parameters('siteName'), '/staging')]",
    "location": "[variables('location')]",
    "properties": {
      "buildVersion": "[parameters('sites_buildVersion')]"
    }
  },
  {
    "type": "Microsoft.Web/sites",
    "apiVersion": "2018-02-01",
    "name": "[parameters('siteName')]",
    "location": "[variables('location')]",
    "dependsOn": [
      "[resourceId('Microsoft.Web/sites/slots', parameters('siteName'), 'staging')]"
    ],
    "properties": {
      "targetBuildVersion": "[parameters('sites_buildVersion')]"
    }
  }
]
```

**Note:** It’s crucial for the second resource (`Microsoft.Web/sites`) to depend on the slot resource to ensure the slot is marked with the `buildVersion` before the swap occurs.

## Combo Swap with New Version Deployment

You can also deploy new application binaries to the slot and automatically swap them afterward. Although this approach skips the test of the new versions in the staging slot, you can quickly revert to the previous version if something goes wrong.

Here’s how to update the slot template to include new version deployment and swapping:

```jsonc
{
  "type": "Microsoft.Web/sites/slots",
  "apiVersion": "2018-02-01",
  "name": "[concat(parameters('siteName'), '/staging')]",
  "location": "[variables('location')]",
  "properties": {
    "buildVersion": "[parameters('sites_buildVersion')]"
  },
  "resources": [
    {
      "type": "Extensions",
      "apiVersion": "2022-09-01",
      "name": "onedeploy",
      "properties": {
        "packageUri": "https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v2.zip",
        "type": "zip"
      },
      "dependsOn": [
        "[concat('Microsoft.Web/sites/', parameters('siteName'), '/slots/staging')]"
      ]
    }
  ]
}
```

This updated template includes both the deployment of the new application package and the setup for swapping the staging slot.

## Conclusion

You’ve now learned how to swap App Service slots using ARM templates, including how to manage build versions and automate the process of deploying and swapping slots. This approach provides a robust way to test and deploy updates with minimal downtime and risk.

Feel free to leave any questions or comments by using `Issues`.
