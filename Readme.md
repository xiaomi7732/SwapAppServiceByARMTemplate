# Deploy to ARM slot and swap

## Publish the website locally

```shell
dotnet publish -c Release
```

```shell
MyAPI C:\AIR\DeployAppServiceBreakDown\src\MyAPI\bin\Release\net8.0\publish\
```

```shell
Compress-Archive C:\AIR\DeployAppServiceBreakDown\src\MyAPI\bin\Release\net8.0\publish\* C:\AIR\DeployAppServiceBreakDown\src\MyAPI\bin\Release\net8.0\publish\v1.zip
```

