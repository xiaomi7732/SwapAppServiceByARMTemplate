# Swap App Service Slots with ARM Templates: A Step-by-Step Guide

In my previous post, we covered [how to deploy an application to an app service](./Readme.md), and [how to deploy an application to a slot](./DeployToSlot.md). In this post, we will look into how to swap them.

Swap slots by ARM template could be mind boggling at the beginning. This post intend to clear it up. If you prefer to look into the templates on your own, check out [swap.json](./ARM/swap.json) and [swap-combo.json](./ARM/swap-combo.json). Assuming you have deployed an app with a slot, here's an abstraction of how the ARM template works:

1. It set an arbitrary build version to the slot, for example, `Ver.aaa`.
2. It updates the `Production` with a `targetBuildVersion`, to the same version of `Ver.aaa`.

Why? Once a template like that is submitted, ARM will follow the instructions above by:

1. Mark slot with the build version of `Ver.aaa`;
2. Swap the current Production with any slow that with the version - oh, and that happened to be the slot in step 1;

So, a minimal swap template will look like this:

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

It is important for the sites deployment (the 2nd resource in the template above) to depends on the slot, so that the slot portion got deployed before the sites. That is to making sure that the slot is marked `Ver.aaa` before the swap, which is going to seek for a slot with the given `buildVersion`, happens.

And that's how to do a swap by using ARM template.

## Combo swap with new version deployment

Putting things together, it is possible to deploy the newer binaries of the application into the slow, and auto-swap right after it.

One question to that approach is why slot then? Well, to the least, you will have the capability to swap back if something goes south.

Here's the updated portion of the slot template:

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
},
```
