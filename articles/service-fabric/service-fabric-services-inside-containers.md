---
title: How to containerize your Azure Service Fabric microservices (preview)
description: Azure Service Fabric has added new functionality to containerize your Service Fabric microservices. This feature is currently in preview.
services: service-fabric
documentationcenter: .net
author: anmolah
manager: anmolah
editor: 'anmolah'

ms.assetid: 0b41efb3-4063-4600-89f5-b077ea81fa3a
ms.service: service-fabric
ms.devlang: dotNet
ms.topic: article
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 8/04/2017
ms.author: anmola
---
# How to containerize your Service Fabric Reliable Services and Reliable Actors (Preview)

Service Fabric supports containerizing Service Fabric microservices (Reliable Services, and Reliable Actor based services). For more information, see [service fabric containers](service-fabric-containers-overview.md).


 This feature is in preview and this article provides the various steps to get your service running inside a container.  

> [!NOTE]
> This feature is in preview and is not supported in production. Currently this feature only works for Windows.

## Steps to containerize your Service Fabric Application

1. Open your Service Fabric application in Visual Studio.

2. Add class [SFBinaryLoader.cs](https://github.com/Azure/service-fabric-scripts-and-templates/blob/master/code/SFBinaryLoaderForContainers/SFBinaryLoader.cs) to your project. The code in this class is a helper to correctly load the Service Fabric runtime binaries inside your application when running inside a container.

3. For each code package you would like to containerize, initialize the loader at the program entry point. Add the static constructor shown in the following code snippet to your program entry point file.

  ```csharp
  namespace MyApplication
  {
      internal static class Program
      {
          static Program()
          {
              SFBinaryLoader.Initialize();
          }

          /// <summary>
          /// This is the entry point of the service host process.
          /// </summary>
          private static void Main()
          {
  ```

4. Build and [package](service-fabric-package-apps.md#Package-App) your project. To build and create a package, right-click the application project in Solution Explorer and choose the **Package** command.

5. For every code package you need to containerize, run the PowerShell script [CreateDockerPackage.ps1](https://github.com/Azure/service-fabric-scripts-and-templates/blob/master/scripts/CodePackageToDockerPackage/CreateDockerPackage.ps1). The usage is as follows:
  ```powershell
    $codePackagePath = 'Path to the code package to containerize.'
    $dockerPackageOutputDirectoryPath = 'Output path for the generated docker folder.'
    $applicationExeName = 'Name of the ode package executable.'
    CreateDockerPackage.ps1 -CodePackageDirectoryPath $codePackagePath -DockerPackageOutputDirectoryPath $dockerPackageOutputDirectoryPath -ApplicationExeName $applicationExeName
 ```
  The script creates a folder with Docker artifacts at $dockerPackageOutputDirectoryPath. Modify the generated Dockerfile to expose any ports, run setup scripts etc. based on your needs.

6. Next you need to [build](service-fabric-get-started-containers.md#Build-Containers) and [push](service-fabric-get-started-containers.md#Push-Containers) your Docker container package to your repository.

7. Modify the ApplicationManifest.xml and ServiceManifest.xml to add your container image, repository information, registry authentication, and port-to-host mapping. For modifying the manifests, see [Create an Azure Service Fabric container application](service-fabric-get-started-containers.md). The code package definition in the service manifest needs to be replaced with corresponding container image. Make sure to change the EntryPoint to a ContainerHost type.

  ```xml
<!-- Code package is your service executable. -->
<CodePackage Name="Code" Version="1.0.0">
  <EntryPoint>
    <!-- Follow this link for more information about deploying Windows containers to Service Fabric: https://aka.ms/sfguestcontainers -->
    <ContainerHost>
      <ImageName>myregistry.azurecr.io/samples/helloworldapp</ImageName>
    </ContainerHost>
  </EntryPoint>
  <!-- Pass environment variables to your container: -->    
</CodePackage>
  ```

8. Add the port-to-host mapping for your replicator and service endpoint. Since both these ports are assigned at runtime by Service Fabric, the ContainerPort is set to zero to use the assigned port for mapping.

 ```xml
<Policies>
  <ContainerHostPolicies CodePackageRef="Code">
    <PortBinding ContainerPort="0" EndpointRef="ServiceEndpoint"/>
    <PortBinding ContainerPort="0" EndpointRef="ReplicatorEndpoint"/>
  </ContainerHostPolicies>
</Policies>
 ```

9. To test this application, you need to deploy it to a cluster that is running version 5.7 or higher. In addition, you need to edit and update the cluster settings to enable this preview feature. Follow the steps in this [article](service-fabric-cluster-fabric-settings.md) to add the setting shown next.
```
      {
        "name": "Hosting",
        "parameters": [
          {
            "name": "FabricContainerAppsEnabled",
            "value": "true"
          }
        ]
      }
```
10. Next [deploy](service-fabric-deploy-remove-applications.md) the edited application package to this cluster.

You should now have a containerized Service Fabric application running your cluster.

## Next steps
* Learn more about running [containers on Service Fabric](service-fabric-get-started-containers.md).
* Learn about the Service Fabric [application life-cycle](service-fabric-application-lifecycle.md).
