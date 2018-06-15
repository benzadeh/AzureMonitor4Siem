# Integrate Azure Monitor logs with a SIEM, Analytics Tool, or Monitoring Solution

AzureMonitor4Siem makes it easy to setup [Azure Monitor](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-azure-monitor) and download the Azure activity logs it provides to a Windows, macOS, or Linux computer.  From there, you can quickly setup a log file-based integration with a SIEM, analytics tool, or monitoring solution of your choice.  

Want to do more?  No problem!  Feel free to use our project and source code as a starter kit.   

## How It Works
Azure Monitor enables you to [stream Azure activity logs](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-stream-activity-logs-event-hubs) into an [Azure Event Hub](https://azure.microsoft.com/en-us/services/event-hubs/).  A client application can then connect to Event Hub and download the activity logs placed into it.  

Our AzureMonitor4Siem project has two components: a script to setup the resources in Azure to support log streaming, and an application to download the logs to a computer.

***setupAzureMonitor&#46;sh*** is a Bash shell script that automates the creation and configuration of the Azure resources required to support activity log streaming from Azure Monitor into Event Hub.  Our Azure CLI powered script:

* Creates an Azure Resource Group that will contain all the Azure-based resources required to support the integration
* Creates an Event Hub namespace 
* Configures Azure Monitor to export and stream activity logs into a new Event Hub within the just-created Event Hub namespace 
* Creates and configures an Azure Blob Storage account and Storage Container, which will be used to manage the synchronization state of the client application
* Generates Shared Access Signatures for both the Event Hub and Blob Storage Container, eliminating the need to use master account keys in the client application
* Generates a configuration file (***azureSettings&#46;json***) storing all connection parameters required by the client application.

***AzureMonitor4Siem***, the client application, is built using [.NET Core](https://www.microsoft.com/net/) and can be run on Windows, macOS, and Linux.  It connects to the Event Hub created by Azure Monitor and downloads Azure activity logs to the local filesystem of the computer it's running on.  This application makes use of:

*  [Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host) feaure introduced in .NET Core 2.1 
*  [Event Hub Processor Host](https://www.nuget.org/packages/Microsoft.Azure.ServiceBus.EventProcessorHost), part of the Event Hub .NET SDK, which integrates with Azure Storage to support checkpointing and parallel receives
*  Connectivity values within ***azureSettings&#46;json*** configuration file as generated by the ***setupAzureMonitor&#46;sh*** script

Let's get started! 

## Setup a Development Environment
*  Clone this GitHub repository
*  You will need a Bash environment and Azure CLI 2.0 to run the ***setupAzureMonitor&#46;sh*** setup script
   *  Mac and Linux environments inclue Bash
      * [Download](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) and install Azure CLI 2.0.
   *  Windows Server and Windows 10
      * [Windows 10 Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) and [Windows Server Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-on-server), known as "WSL", provide a Bash environment that can run in a Windows 10 and Windows Server environment
      * If you're using WSL, you'll need to install the Linux version of Azure CLI.
      *  [Download](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) and install Azure CLI 2.0.
   *  Alternatively, ***setupAzureMonitor&#46;sh*** does not have to run on the same computer as the client application.  You can run it elsewhere, as long as that environment has Bash and Azure CLI installed.
      * [Azure Cloud Shell](https://shell.azure.com) provides you a browser-based Bash environment with Azure CLI pre-installed.   
      * You can also [deploy](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal) a Linux Virtual machine in Azure

*  [Download](https://www.microsoft.com/net/download/core) and install .NET Core 2.1 SDK, which is necessary to build and run the ***AzureMonitor4Siem*** application that downloads logs from Azure your computer


### Identify a User Account and Azure Environment
*  You'll need an Azure account that has the privledges to create and configure the Azure resources and services described above.  
* If you don’t already have an Azure account, [register for a free trial](https://azure.microsoft.com/en-us/free/) that includes $200 in credits.

## Review and Run setupAzureMonitor&#46;sh

*  Go to the folder containing your local clone of this repository
*  In a text editor, open ***setupAzureMonitor&#46;sh***  
*  There are a number of variables you may wish to alter.  There are some default values provided that you can probably use as-is - but let's be sure!  
      * ***AZ_RESOURCE_GROUP*** is the name of the Azure Resource group that will contain the Azure resources created and used.  The name you use must be unique to your Azure subscription.
      *  ***AZ_REGION*** is the name of the Azure region in which the Resource Group required for the integration will be created, and all of the supporting resources will be deployed.  By default, the script deploys to "westus2".  You can determine the list of region names through the Azure CLI by logging in and listing them as follows:
         * az login
         * az account list-locations --query [*].name
   *  Two variables must be unique across all of Azure.  To help you, we automatically append the current date and time, using a variable called ***DATE_TIME*** to the following values
      * ***AZ_STORAGE_ACCOUNT*** - the name of the Azure Storage account to create
      * ***AZ_EVENTHUB_NAMESPACE***  - the name of the Azure Event Hub namespace
*  Optional: If you need to run ***setupAzureMonitor&#46;sh*** elsewhere (i.e. your computer doesn't have Bash), copy it to the destination environment. You can also consider copying-and-pasting its content to a file of the same name.
*  Change the permissions on ***setupAzureMonitor&#46;sh*** to make sure it can be run:
   * chmod +x ***setupAzureMonitor&#46;sh***
* Run ***setupAzureMonitor&#46;sh*** as follows:
   * ./setupAzureMonitor.sh
      *  Follow the directions to log in to Azure.  If you have logged in to Azure using Azure CLI before ("az login"), you have the option of using locally cached credentials.  
      *  You will be given the option of destroying any resource group that has the same name as the one you wish to use.  Deleting the resource group will also destroy all resources within it.  This is particularly useful if you plan to run the ***setupAzureMonitor&#46;sh*** script multiple times for testing and demo purposes.
*  The script will take 5 to 10 minutes to complete
   * You may see "Waiting for Event Hub Creation to complete" several times
*  You'll be given the option of identifying the folder where downloaded activity log files are stored.  
   *  If you change your mind, alter the value of ***az_local_logs_dir*** in ***azureSettings&#46;json***.
   *  If you don't provide a value, we'll create a temporary folder each time you start the application
* A file named ***azureSettings&#46;json*** will be created containing all of the connection parameters needed by ***AzureMonitor4Siem*** to download Azure activity logs. 
   *  If you are running ***setupAzureMonitor&#46;sh*** in a different environment than where you will build and run AzureMonitor4Siem, you must copy  ***azureSettings&#46;json*** back to it.  Alternatvely, you can copy-and-paste the content into a file of the same name.


## Build and Run AzureMonitor4Siem
*  Navigate to the folder containing your local clone of this repository
*  Make sure that ***azureSettings&#46;json***, as created by ***setupAzureMonitor&#46;sh***  is present
*  Run these commands to build and run the client application that will download Activity logs to your computer:
    *  dotnet clean
    *  dotnet build
    *  dotnet run

* Upon successful startup, you should see something like this:

```
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      Log directory is /path/you/identified info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      OnStarted has been called.
Application started. Press Ctrl+C to shut down.
Hosting environment: Development
Content root path: /path/to/your/cloned/repo/AzureMonitor4Siem/bin/Debug/netcoreapp2.1/
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      SimpleEventProcessor initialized. Partition: '0'
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      SimpleEventProcessor initialized. Partition: '1'
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      SimpleEventProcessor initialized. Partition: '2'
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      SimpleEventProcessor initialized. Partition: '3'
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
To exit the program, simpy type Control-C
```

Whenever ***AzureMonitor4Siem*** downloads a new log file from Event Hub, you'll see a notice similiar to this on your screen

```
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
File written in /path/you/identified/1528870414.74773 at 6/12/18 11:13:34 PM
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
File written in /path/you/identified/1528870414.74775 at 6/12/18 11:13:34 PM
```

Pressing Control-C will shut down AzureMonitor4Siem.   
```
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      OnStopping has been called.
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      Processor Shutting Down. Partition '1', Reason: 'Shutdown'.
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      Processor Shutting Down. Partition '3', Reason: 'Shutdown'.
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      Processor Shutting Down. Partition '0', Reason: 'Shutdown'.
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      Processor Shutting Down. Partition '2', Reason: 'Shutdown'.
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      Log directory is /path/you/identified
info: AzureMonitor4Siem.LifetimeEventsHostedService[0]
      OnStopped has been called.
```

## Publish and Install AzureMonitor4Siem

If you're happy with the results, you can optionally publish AzureMonitor4Siem into a self-contained application.  This will enable you to run AzureMonitor4Siem from another folder or  another computer of the same operating system.
*  Navigate to the folder containing your local clone of this repository
*  Determine the directory you want to publish and install AzureMonitor4Siem into.
*  Find the "Runtime Identifier" (RID) that corresponds to your operating system environment on the following website https://docs.microsoft.com/en-us/dotnet/core/rid-catalog
   * Here are the RIDs for some popular operating systems
      * Windows 10 / Windows Server 2016 is win10-x64
      * Mac OS X is osx-x64
      * Ubuntu is ubuntu-x64
* Run the dotnet publish command, supply the RID for your environment, and your installation folder
   * dotnet publish -c Release --self-contained -r your-RID -o /your/destination/folder
   * Here's an example for Ubuntu
      * dotnet publish -c Release --self-contained -r ubuntu-x64 -o /your/destination/folder
   * The publish command will copy your configuration file ***azureSettings&#46;json*** to the destination folde as well.
   * Now, you can run AzureMonitor4Siem simply like this:
      * /your/destination/folder/AzureMonitor4Siem

## Customizing AzureMonitor4Siem

***LifetimeEventsHostedService.cs*** contains the code that manages the lifecycle of the ***AzureMonitor4Siem***.  There are event handlers where you can add additional activities that occur upon startup (OnStarted), during shut down (OnStopping), and after shut down (OnStop). 

***SimpleEventProcessor.cs*** manages the processing of the activity logs placed into Azure Event Hub.  It is registered within ***LifetimeEventsHostedService*** inside OnStarted and un-registerered through OnStopping.  Within ***SimpleEventProcessor*** ***ProcessEventsAsync*** is responsible for iterating over the Azure activity logs placed by Azure Monitor into Event Hub and writing them to a local file.   As such, ***ProcessEventsAsync*** is a great place to author your own custom action.   


## Acknowledgements

AzureMonitor4Siem is based upon the code contained in the following GitHub repositories.   Thank you the individuals involved in creating and providing Open Source solutions on Azure.

* [Receive events with the Event Processor Host in .NET Standard](https://github.com/Azure/azure-event-hubs/tree/master/samples/DotNet/Microsoft.Azure.EventHubs/SampleEphReceiver)
* [ASP.NET Generic Host Sample](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/host/generic-host/samples/2.x/GenericHostSample)


## Questions and comments
We'd love to get your feedback about this sample. You can send your questions and suggestions to us in the Issues section of this repository.

Questions about Azure Monitor development in general should be posted to Stack Overflow. Make sure that your questions or comments are tagged with [azure-monitoring](https://stackoverflow.com/questions/tagged/azure-monitoring).

## Resources

*  [Azure Monitor](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-azure-monitor)
*  [Azure Event Hubs](https://azure.microsoft.com/en-us/services/event-hubs/)
*  [Azure Storage](https://azure.microsoft.com/en-us/services/storage/)
*  [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
*  [.NET Core](https://www.microsoft.com/net/)

## Copyright

Copyright (c) 2018 Tam Huynh. All rights reserved. 


### Disclaimer ###
**THIS CODE IS PROVIDED *AS IS* WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESS OR IMPLIED, INCLUDING ANY IMPLIED WARRANTIES OF FITNESS FOR A PARTICULAR PURPOSE, MERCHANTABILITY, OR NON-INFRINGEMENT.**