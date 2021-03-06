---
title: Deploy and upgrade Azure Service Fabric services locally | Microsoft Docs
description: Learn how to set up a local Service Fabric cluster, deploy an existing application to it, and then upgrade that application.
services: service-fabric
documentationcenter: .net
author: rwike77
manager: timlt
editor: ''

ms.assetid: 60a1f6a5-5478-46c0-80a8-18fe62da17a8
ms.service: service-fabric
ms.devlang: dotNet
ms.topic: conceptual
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 11/03/2017
ms.author: ryanwi

---
# Get started with deploying and upgrading applications on your local cluster
The Azure Service Fabric SDK includes a full local development environment that you can use to quickly get started with deploying and managing applications on a local cluster. In this article, you create a local cluster, deploy an existing application to it, and then upgrade that application to a new version, all from Windows PowerShell.

> [!NOTE]
> This article assumes that you already [set up your development environment](service-fabric-get-started.md).
> 
> 

## Create a local cluster
A Service Fabric cluster represents a set of hardware resources that you can deploy applications to. Typically, a cluster is made up of anywhere from five to many thousands of machines. However, the Service Fabric SDK includes a cluster configuration that can run on a single machine.

It is important to understand that the Service Fabric local cluster is not an emulator or simulator. It runs the same platform code that is found on multi-machine clusters. The only difference is that it runs the platform processes that are normally spread across five machines on one machine.

The SDK provides two ways to set up a local cluster: a Windows PowerShell script and the Local Cluster Manager system tray app. In this tutorial, we use the PowerShell script.

> [!NOTE]
> If you have already created a local cluster by deploying an application from Visual Studio, you can skip this section.
> 
> 

1. Launch a new PowerShell window as an administrator.
2. Run the cluster setup script from the SDK folder:
   
    ```powershell
    & "$ENV:ProgramFiles\Microsoft SDKs\Service Fabric\ClusterSetup\DevClusterSetup.ps1"
    ```
   
    Cluster setup takes a few moments. After setup is finished, you should see output similar to:
   
    ![Cluster setup output][cluster-setup-success]
   
    You are now ready to try deploying an application to your cluster.

## Deploy an application
The Service Fabric SDK includes a rich set of frameworks and developer tooling for creating applications. If you are interested in learning how to create applications in Visual Studio, see [Create your first Service Fabric application in Visual Studio](service-fabric-create-your-first-application-in-visual-studio.md).

In this tutorial, you use an existing sample application (called WordCount) so that you can focus on the management aspects of the platform: deployment, monitoring, and upgrade.

1. Launch a new PowerShell window as an administrator.
2. Import the Service Fabric SDK PowerShell module.
   
    ```powershell
    Import-Module "$ENV:ProgramFiles\Microsoft SDKs\Service Fabric\Tools\PSModule\ServiceFabricSDK\ServiceFabricSDK.psm1"
    ```
3. Create a directory to store the application that you download and deploy, such as C:\ServiceFabric.
   
    ```powershell
    mkdir c:\ServiceFabric\
    cd c:\ServiceFabric\
    ```
4. [Download the WordCount application](http://aka.ms/servicefabric-wordcountapp) to the location you created.  Note: the Microsoft Edge browser saves the file with a *.zip* extension.  Change the file extension to *.sfpkg*.
5. Connect to the local cluster:
   
    ```powershell
    Connect-ServiceFabricCluster localhost:19000
    ```
6. Create a new application using the SDK's deployment command with a name and a path to the application package.
   
    ```powershell  
   Publish-NewServiceFabricApplication -ApplicationPackagePath c:\ServiceFabric\WordCountV1.sfpkg -ApplicationName "fabric:/WordCount"
    ```
   
    If all goes well, you should see the following output:
   
    ![Deploy an application to the local cluster][deploy-app-to-local-cluster]
7. To see the application in action, launch the browser and navigate to [http://localhost:8081/wordcount/index.html](http://localhost:8081/wordcount/index.html). You should see:
   
    ![Deployed application UI][deployed-app-ui]
   
    The WordCount application is simple. It includes client-side JavaScript code to generate random five-character "words", which are then relayed to the application via ASP.NET Web API. A stateful service tracks the number of words counted. They are partitioned based on the first character of the word. You can find the source code for the WordCount app in the [classic getting started samples](https://github.com/Azure-Samples/service-fabric-dotnet-getting-started/tree/classic/Services/WordCount).
   
    The application that we deployed contains four partitions. So words beginning with A through G are stored in the first partition, words beginning with H through N are stored in the second partition, and so on.

## View application details and status
Now that we have deployed the application, let's look at some of the app details in PowerShell.

1. Query all deployed applications on the cluster:
   
    ```powershell
    Get-ServiceFabricApplication
    ```
   
    Assuming that you have only deployed the WordCount app, you see something similar to:
   
    ![Query all deployed applications in PowerShell][ps-getsfapp]
2. Go to the next level by querying the set of services that are included in the WordCount application.
   
    ```powershell
    Get-ServiceFabricService -ApplicationName 'fabric:/WordCount'
    ```
   
    ![List services for the application in PowerShell][ps-getsfsvc]
   
    The application is made up of two services, the web front end, and the stateful service that manages the words.
3. Finally, look at the list of partitions for WordCountService:
   
    ```powershell
    Get-ServiceFabricPartition 'fabric:/WordCount/WordCountService'
    ```
   
    ![View the service partitions in PowerShell][ps-getsfpartitions]
   
    The set of commands that you used, like all Service Fabric PowerShell commands, are available for any cluster that you might connect to, local or remote.
   
    For a more visual way to interact with the cluster, you can use the web-based Service Fabric Explorer tool by navigating to [http://localhost:19080/Explorer](http://localhost:19080/Explorer) in the browser.
   
    ![View application details in Service Fabric Explorer][sfx-service-overview]
   
   > [!NOTE]
   > To learn more about Service Fabric Explorer, see [Visualizing your cluster with Service Fabric Explorer](service-fabric-visualizing-your-cluster.md).
   > 
   > 

## Upgrade an application
Service Fabric provides no-downtime upgrades by monitoring the health of the application as it rolls out across the cluster. Perform an upgrade of the WordCount application.

The new version of the application now counts only words that begin with a vowel. As the upgrade rolls out, we see two changes in the application's behavior. First, the rate at which the count grows should slow, since fewer words are being counted. Second, since the first partition has two vowels (A and E) and all other partitions contain only one each, its count should eventually start to outpace the others.

1. [Download the WordCount version 2 package](http://aka.ms/servicefabric-wordcountappv2) to the same location where you downloaded the version 1 package.
2. Return to your PowerShell window and use the SDK's upgrade command to register the new version in the cluster. Then begin upgrading the fabric:/WordCount application.
   
    ```powershell
    Publish-UpgradedServiceFabricApplication -ApplicationPackagePath C:\ServiceFabric\WordCountV2.sfpkg -ApplicationName "fabric:/WordCount" -UpgradeParameters @{"FailureAction"="Rollback"; "UpgradeReplicaSetCheckTimeout"=1; "Monitored"=$true; "Force"=$true}
    ```
   
    You should see the following output in PowerShell as the upgrade begins.
   
    ![Upgrade progress in PowerShell][ps-appupgradeprogress]
3. While the upgrade is proceeding, you may find it easier to monitor its status from Service Fabric Explorer. Launch a browser window and navigate to [http://localhost:19080/Explorer](http://localhost:19080/Explorer). Expand **Applications** in the tree on the left, then choose **WordCount**, and finally **fabric:/WordCount**. In the essentials tab, you see the status of the upgrade as it proceeds through the cluster's upgrade domains.
   
    ![Upgrade progress in Service Fabric Explorer][sfx-upgradeprogress]
   
    As the upgrade proceeds through each domain, health checks are performed to ensure that the application is behaving properly.
4. If you rerun the earlier query for the set of services in the fabric:/WordCount application, notice that the WordCountService version changed but the WordCountWebService version did not:
   
    ```powershell
    Get-ServiceFabricService -ApplicationName 'fabric:/WordCount'
    ```
   
    ![Query application services after upgrade][ps-getsfsvc-postupgrade]
   
    This example highlights how Service Fabric manages application upgrades. It touches only the set of services (or code/configuration packages within those services) that have changed, which makes the process of upgrading faster and more reliable.
5. Finally, return to the browser to observe the behavior of the new application version. As expected, the count progresses more slowly, and the first partition ends up with slightly more of the volume.
   
    ![View the new version of the application in the browser][deployed-app-ui-v2]

## Cleaning up
Before wrapping up, it's important to remember that the local cluster is real. Applications continue to run in the background until you remove them.  Depending on the nature of your apps, a running app can take up significant resources on your machine. You have several options to manage applications and the cluster:

1. To remove an individual application and all it's data, run the following command:
   
    ```powershell
    Unpublish-ServiceFabricApplication -ApplicationName "fabric:/WordCount"
    ```
   
    Or, delete the application from the Service Fabric Explorer **ACTIONS** menu or the context menu in the left-hand application list view.
   
    ![Delete an application is Service Fabric Explorer][sfe-delete-application]
2. After deleting the application from the cluster, unregister versions 1.0.0 and 2.0.0 of the WordCount application type. Deletion removes the application packages, including the code and configuration, from the cluster's image store.
   
    ```powershell
    Remove-ServiceFabricApplicationType -ApplicationTypeName WordCount -ApplicationTypeVersion 2.0.0
    Remove-ServiceFabricApplicationType -ApplicationTypeName WordCount -ApplicationTypeVersion 1.0.0
    ```
   
    Or, in Service Fabric Explorer, choose **Unprovision Type** for the application.
3. To shut down the cluster but keep the application data and traces, click **Stop Local Cluster** in the system tray app.
4. To delete the cluster entirely, click **Remove Local Cluster** in the system tray app. This option will result in another slow deployment the next time you press F5 in Visual Studio. Remove the local cluster only if you don't intend to use it for some time or if you need to reclaim resources.

## One-node and five-node cluster mode
When developing applications, you often find yourself doing quick iterations of writing code, debugging, changing code, and debugging. To help optimize this process, the local cluster can run in two modes: one-node or five-node. Both cluster modes have their benefits. Five-node cluster mode enables you to work with a real cluster. You can test failover scenarios, work with more instances and replicas of your services. One-node cluster mode is optimized to do quick deployment and registration of services, to help you quickly validate code using the Service Fabric runtime.

Neither one-node cluster or five-node cluster modes are an emulator or simulator. The local development cluster runs the same platform code that is found on multi-machine clusters.

> [!WARNING]
> When you change the cluster mode, the current cluster is removed from your system and a new cluster is created. The data stored in the cluster is deleted when you change cluster mode.
> 
> 

To change the mode to one-node cluster, select **Switch Cluster Mode** in the Service Fabric Local Cluster Manager.

![Switch cluster mode][switch-cluster-mode]

Or, change the cluster mode using PowerShell:

1. Launch a new PowerShell window as an administrator.
2. Run the cluster setup script from the SDK folder:
   
    ```powershell
    & "$ENV:ProgramFiles\Microsoft SDKs\Service Fabric\ClusterSetup\DevClusterSetup.ps1" -CreateOneNodeCluster
    ```
   
    Cluster setup takes a few moments. After setup is finished, you should see output similar to:
   
    ![Cluster setup output][cluster-setup-success-1-node]

## Next steps
* Now that you have deployed and upgraded some pre-built applications, you can [try building your own in Visual Studio](service-fabric-create-your-first-application-in-visual-studio.md).
* All the actions performed on the local cluster in this article can be performed on an [Azure cluster](service-fabric-cluster-creation-via-portal.md) as well.
* The upgrade that we performed in this article was basic. See the [upgrade documentation](service-fabric-application-upgrade.md) to learn more about the power and flexibility of Service Fabric upgrades.

<!-- Images -->

[cluster-setup-success]: ./media/service-fabric-get-started-with-a-local-cluster/LocalClusterSetup.png
[extracted-app-package]: ./media/service-fabric-get-started-with-a-local-cluster/ExtractedAppPackage.png
[deploy-app-to-local-cluster]: ./media/service-fabric-get-started-with-a-local-cluster/DeployAppToLocalCluster.png
[deployed-app-ui]: ./media/service-fabric-get-started-with-a-local-cluster/DeployedAppUI-v1.png
[deployed-app-ui-v2]: ./media/service-fabric-get-started-with-a-local-cluster/DeployedAppUI-PostUpgrade.png
[sfx-app-instance]: ./media/service-fabric-get-started-with-a-local-cluster/SfxAppInstance.png
[sfx-two-app-instances-different-partitions]: ./media/service-fabric-get-started-with-a-local-cluster/SfxTwoAppInstances-DifferentPartitionCount.png
[ps-getsfapp]: ./media/service-fabric-get-started-with-a-local-cluster/PS-GetSFApp.png
[ps-getsfsvc]: ./media/service-fabric-get-started-with-a-local-cluster/PS-GetSFSvc.png
[ps-getsfpartitions]: ./media/service-fabric-get-started-with-a-local-cluster/PS-GetSFPartitions.png
[ps-appupgradeprogress]: ./media/service-fabric-get-started-with-a-local-cluster/PS-AppUpgradeProgress.png
[ps-getsfsvc-postupgrade]: ./media/service-fabric-get-started-with-a-local-cluster/PS-GetSFSvc-PostUpgrade.png
[sfx-upgradeprogress]: ./media/service-fabric-get-started-with-a-local-cluster/SfxUpgradeOverview.png
[sfx-service-overview]: ./media/service-fabric-get-started-with-a-local-cluster/sfx-service-overview.png
[sfe-delete-application]: ./media/service-fabric-get-started-with-a-local-cluster/sfe-delete-application.png
[cluster-setup-success-1-node]: ./media/service-fabric-get-started-with-a-local-cluster/cluster-setup-success-1-node.png
[switch-cluster-mode]: ./media/service-fabric-get-started-with-a-local-cluster/switch-cluster-mode.png
