# Deploy to ARM slot and swap

## Publish the website locally

```shell
dotnet publish -c Release
```

```shell
MyAPI C:\AIR\DeployAppServiceBreakDown\src\MyAPI\bin\Release\net8.0\publish\
```

```shell
Compress-Archive C:\AIR\DeployAppServiceBreakDown\src\MyAPI\bin\Release\net8.0\publish\* ..\..\out\v1.zip
```

```shell
Compress-Archive C:\AIR\DeployAppServiceBreakDown\src\MyAPI\bin\Release\net8.0\publish\* ..\..\out\v2.zip
```

Download urls

```shell
https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v1.zip
https://xm7732public.blob.core.windows.net/public/arm-swap-examples/v2.zip
```

```shell
$rgName="Swap-example-2"
az group create -n $rgName --location "West US"
```

```json
az deployment group create -n swapdemo -g $rgName --template-file .\initial.json
az deployment group create -n swapdemo -g $rgName --template-file .\swap.json
```