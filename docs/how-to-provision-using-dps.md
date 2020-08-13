---
# Mandatory fields.
title: Azure Digital Twins auto-provisiong using Device Provisioning Service
titleSuffix: Azure Digital Twins
description: See how to set up automated process to provision and retire IoT devices in Azure Digital Twins using Device Provisioning Service.
author: michielvanschaik
ms.author: mivansch # Microsoft employees only
ms.date: 8/13/2020
ms.topic: how-to
ms.service: digital-twins

# Optional fields. Don't forget to remove # if you need a field.
# ms.custom: can-be-multiple-comma-separated
# ms.reviewer: MSFT-alias-of-reviewer
# manager: MSFT-alias-of-manager-or-PM-counterpart
---

# Azure Digital Twins auto-provisiong using Device Provisioning Service

In this article, you'll learn how to integrate Azure Digital Twins with [Device Provisioning Service (DPS)](https://docs.microsoft.com/en-us/azure/iot-dps/about-iot-dps).

The solution described in this article will allow you to automate the process to _provision_ and _retire_ IoT Hub devices in Azure Digital Twins using Device Provisioning Service. The better understand a set of general device management stages that are common to all enterprise IoT projects, see [Device Lifecycle](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-device-management-overview#device-lifecycle).

## Prerequisites

Before you can set up the provisioning, you need to have an **Azure Digital Twins instance**. 

If you do not have this set up already, you can create it by following the Azure Digital Twins [*Tutorial: Connect an end-to-end solution*](./tutorial-end-to-end.md). The tutorial will walk you through setting up an Azure Digital Twins instance and an Azure IoT Hub.

## Solution architecture

The image below illustrates both the device provision and retire flow.

![A view of Azure services in an end-to-end scenario, highlighting Device Provisioning Service](media/flows.png)

## Auto-provision device using Device Provisioning Service

You will be attaching Device Provisioning Service to Azure Digital Twins to auto-provision devices through the path below.

![Provision device flow](media/provision.png)

Process flow description.
1. Device contacts the DPS endpoint, passing identifying information to prove its identity.
2. DPS validates device identity by validating the registration ID and key against the enrollment list and calls an Azure Function to do the allocation.
3. The Azure Function creates a new twin in Azure Digital Twins for the device.
4. DPS registers the device with an IoT hub and populates the device's desired twin state.
5. The IoT hub returns device ID information  and the IoT hub connection information to the device. The device can now connect to the IoT hub.

Next are the process steps to setup the auto-provision device flow.

### Create a Device Provisioning Service

When a new device is providioned using Device Provisioning Service, a new twin for that device can be created in Azure Digital Twins.

Create a Device Provisioning Service instance, which will be used to provision IoT devices. You can either use the Azure CLI instructions below, or use the Azure portal: [*Quickstart: Set up the IoT Hub Device Provisioning Service with the Azure portal*](../iot-dps/quick-setup-auto-provision).

```azurecli-interactive
# Create a Device Provisioning Service. Specify a name and region.
az iot dps create --name <Device Provisioning Service name> --resource-group <resource group name> --location <region, for example: East US>
```

### Create an Azure function 

Next, you'll create an Http request-triggered function inside a function app. You can use the function app created in the end-to-end tutorial ([*Tutorial: Connect an end-to-end solution*](./tutorial-end-to-end.md)), or your own. 

This function will be used by the Device Provisioning Service in a [Custom Allocation Policy](../iot-dps/how-to-use-custom-allocation-policies) provisioning a new device. For more information about using Http requests with Azure functions, see [*Azure Http request trigger for Azure Functions*](../azure-functions/functions-bindings-http-webhook-trigger.md).

Inside your published function app, replace the function code with the following code.

```C#
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Devices.Shared;
using Microsoft.Azure.Devices.Provisioning.Service;
using System.Net.Http;
using Azure.Identity;
using Azure.DigitalTwins.Core;
using Azure.Core.Pipeline;
using Azure;
using System.Collections.Generic;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

namespace Samples.AdtIothub
{
    public static class DpsAdtAllocationFunc
    {
        private static string adtAppId = System.Environment.GetEnvironmentVariable("AdtAppId", EnvironmentVariableTarget.Process);
        private static readonly string adtInstanceUrl = System.Environment.GetEnvironmentVariable("AdtInstanceUrl", EnvironmentVariableTarget.Process);
        private static readonly HttpClient httpClient = new HttpClient();

        [FunctionName("DpsAdtAllocationFunc")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req, ILogger log)
        {
            // Get request body
            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            log.LogDebug($"Request.Body: {requestBody}");
            dynamic data = JsonConvert.DeserializeObject(requestBody);

            // Get registration ID of the device
            string regId = data?.deviceRuntimeContext?.registrationId;

            bool fail = false;
            string message = "Uncaught error";
            ResponseObj obj = new ResponseObj();

            // Must have unique registration ID on DPS request 
            if (regId == null)
            {
                message = "Registration ID not provided for the device.";
                log.LogInformation("Registration ID: NULL");
                fail = true;
            }
            else
            {
                string[] hubs = data?.linkedHubs.ToObject<string[]>();

                // Must have hubs selected on the enrollment
                if (hubs == null)
                {
                    message = "No hub group defined for the enrollment.";
                    log.LogInformation("linkedHubs: NULL");
                    fail = true;
                }
                else
                {
                    // Find or create twin based on the provided registration ID and model ID
                    dynamic payloadContext = data?.deviceRuntimeContext?.payload;
                    string dtmi = payloadContext.modelId;
                    log.LogDebug($"payload.modelId: {dtmi}");
                    string dtId = await FindOrCreateTwin(dtmi, regId, log);

                    // Get first linked hub (TODO: select one of the linked hubs based on policy)
                    obj.iotHubHostName = hubs[0];

                    // Specify the initial tags for the device.
                    TwinCollection tags = new TwinCollection();
                    tags["dtmi"] = dtmi;
                    tags["dtId"] = dtId;

                    // Specify the initial desired properties for the device.
                    TwinCollection properties = new TwinCollection();

                    // Add the initial twin state to the response.
                    TwinState twinState = new TwinState(tags, properties);
                    obj.initialTwin = twinState;
                }
            }

            log.LogDebug("Response: " + ((obj.iotHubHostName != null) ? JsonConvert.SerializeObject(obj) : message));

            return (fail)
                ? new BadRequestObjectResult(message)
                : (ActionResult)new OkObjectResult(obj);
        }

        public static async Task<string> FindOrCreateTwin(string dtmi, string regId, ILogger log)
        {
            // Create Digital Twins client
            var cred = new ManagedIdentityCredential(adtAppId);
            var client = new DigitalTwinsClient(new Uri(adtInstanceUrl), cred, new DigitalTwinsClientOptions { Transport = new HttpClientTransport(httpClient) });

            // Find existing twin with registration ID
            string dtId;
            string query = $"SELECT * FROM DigitalTwins T WHERE T.HubRegistrationId = '{regId}' AND IS_OF_MODEL('{dtmi}')";
            AsyncPageable<string> twins = client.QueryAsync(query);
            await foreach (string twinJson in twins)
            {
                // Get DT ID from the Twin
                JObject twin = (JObject)JsonConvert.DeserializeObject(twinJson);
                dtId = (string)twin["$dtId"];
                log.LogInformation($"Twin '{dtId}' with Registration ID '{regId}' found in DT");
                return dtId;
            }

            // Not found, so create new twin
            dtId = regId; // use the Registration ID as the DT ID

            // Define the model type for the twin to be created
            Dictionary<string, object> meta = new Dictionary<string, object>()
            {
                { "$model", dtmi }
            };
            // Initialize the twin properties
            Dictionary<string, object> twinProps = new Dictionary<string, object>()
            {
                { "$metadata", meta },
                { "HubRegistrationId", regId }
            };
            await client.CreateDigitalTwinAsync(dtId, System.Text.Json.JsonSerializer.Serialize<Dictionary<string, object>>(twinProps));
            log.LogInformation($"Twin '{dtId}' created in DT");

            return dtId;
        }
    }

    public class ResponseObj
    {
        public string iotHubHostName { get; set; }
        public TwinState initialTwin { get; set; }
    }
}
```

### Configure your function

Next, you'll need to set environment variables in your function app from earlier, containing the reference to the Azure Digital Twins instance you've created.

You will need the following values from when you set up your instance. 
If you need to gather these values again, use the links below to the corresponding sections in the setup article for finding them in the [Azure portal](https://portal.azure.com).
* Azure Digital Twins instance **_host name_** ([find in portal](../articles/digital-twins/how-to-set-up-instance-portal.md#verify-success-and-collect-important-values))
* Azure AD app registration **_Application (client) ID_** ([find in portal](../articles/digital-twins/how-to-set-up-instance-portal.md#collect-important-values))

```azurecli-interactive
az functionapp config appsettings set --settings "AdtInstanceUrl=https://<Azure Digital Twins instance _host name_> -g <resource group> -n <your App Service (function app) name>"
```

```azurecli-interactive
az functionapp config appsettings set --settings "AdtAppId=<Application (client) ID> -g <resource group> -n <your App Service (function app) name>"
```

### Create Device Provisioning enrollment

Next, you'll need to create an enrollment in Device Provisioning Service using a custom allocation function. How to do this can be found in [Create the enrollment](../iot-dps/how-to-use-custom-allocation-policies#create-the-enrollment.md).

Link the enrollment to the function you just created, by selecting the function in the 'select how you want to assign devices to hubs' field. The enrollment name and primary or secondary SAS key will be used later to configure the device simulator.

### Setting up the device simulator

This sample uses a device simulator that includes provisioning using the Device Provisioning Service. The device simulator is located here: [Azure Digital Twins and IoT Hub Integration Sample](https://github.com/Azure-Samples/digital-twins-iothub-integration/tree/main/device-simulator). Get the sample project on your machine by navigating to the sample link, and selecting the Download ZIP button underneath the title.

The device simulator is based on Node.js version 10.0.x or later. [Prepare your development environment](https://github.com/Azure/azure-iot-sdk-node/blob/master/doc/node-devbox-setup.md) describes how to install Node.js for this tutorial on either Windows or Linux. Go to the device-simulator directory and install the dependencies using the following command.
```cmd
npm install
```

Next, copy the .env.template file to an .env file, and fill in the settings.
```code
PROVISIONING_HOST = "global.azure-devices-provisioning.net"
PROVISIONING_IDSCOPE = "<Device Provisioning Service Scope ID>"
PROVISIONING_REGISTRATION_ID = "<Device Registration ID>"
ADT_MODEL_ID = "<Digital Twins Model ID as registered in Azure Digital Twins>"
PROVISIONING_SYMMETRIC_KEY = "<Device Provisioning Service enrollment primary or secondary SAS key>"
```
### Start running the device simulator

Start the device simulator using the following command in the device-simulator directory.
```cmd
node .\adt_custom_register.js
```

You should see the device being registered and connected to IoT Hub, and then starting to send messages.
![](media/output.png)

### Validate

The device will be automatically registered in Azure Digital Twins. Using the following command to find the Twin of the device in the Azure Digital Twins instance you created.

```azurecli-interactive
az dt twin show -n <Digital Twins instance name> --twin-id <Device Registration ID>"
```

## Auto-retire device using IoT Hub Lifecycle events

You will be attaching IoT Hub Lifecycle events to Azure Digital Twins to auto-retire devices through the path below.

![Retire device flow](media/retire.png)

Process flow description.
1. An external or manual process triggers the deletion of a device in IoT Hub.
2. IoT Hub deletes the device and generates a Device Lifecycle event that will be routed to an Event Hub. 
3. An Azure Function deletes the twin of the device in Azure Digital Twins.

Next are the process steps to setup the auto-retire device flow.

### Create an Event Hub

[todo]

### Create an Azure Function


Next, you'll create an Event Hubs-triggered function inside a function app. You can use the function app created in the end-to-end tutorial ([*Tutorial: Connect an end-to-end solution*](./tutorial-end-to-end.md)), or your own. 

This function will use the IoT Hub Device Lifecycle Events to retire an existing device, see [IoT Hub Non-telemetry events](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-d2c#non-telemetry-events). For more information about using Event Hubs with Azure functions, see [*Azure Event Hubs trigger for Azure Functions*](../azure-functions/functions-bindings-event-hubs-trigger.md).

Inside your published function app, replace the function code with the following code.

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;
using Azure;
using Azure.Core.Pipeline;
using Azure.DigitalTwins.Core;
using Azure.DigitalTwins.Core.Serialization;
using Azure.Identity;
using Microsoft.Azure.EventHubs;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

namespace Samples.AdtIothub
{
    public static class DeleteDeviceInTwinFunc
    {
        private static string adtAppId = System.Environment.GetEnvironmentVariable("AdtAppId", EnvironmentVariableTarget.Process);
        private static readonly string adtInstanceUrl = System.Environment.GetEnvironmentVariable("AdtInstanceUrl", EnvironmentVariableTarget.Process);
        private static readonly HttpClient httpClient = new HttpClient();

        [FunctionName("DeleteDeviceInTwinFunc")]
        public static async Task Run(
            [EventHubTrigger("lifecycleevents", Connection = "EVENTHUB_CONNECTIONSTRING")] EventData[] events, ILogger log)
        {
            var exceptions = new List<Exception>();

            foreach (EventData eventData in events)
            {
                try
                {
                    //log.LogDebug($"EventData: {System.Text.Json.JsonSerializer.Serialize(eventData)}");

                    string opType = eventData.Properties["opType"] as string;
                    if (opType == "deleteDeviceIdentity")
                    {
                        string deviceId = eventData.Properties["deviceId"] as string;
                        //string dtId = eventData.Properties["dtId"] as string; // enriched property not available in 'delete' event

                        // Create Digital Twin client
                        var cred = new ManagedIdentityCredential(adtAppId);
                        var client = new DigitalTwinsClient(new Uri(adtInstanceUrl), cred, new DigitalTwinsClientOptions { Transport = new HttpClientTransport(httpClient) });

                        // Find twin based on the original Regitration ID
                        string regID = deviceId; // simple mapping
                        string dtId = await GetTwinId(client, regID, log);
                        if (dtId != null)
                        {
                            await DeleteRelationships(client, dtId, log);

                            // Delete twin
                            await client.DeleteDigitalTwinAsync(dtId);
                            log.LogInformation($"Twin '{dtId}' deleted in DT");
                        }
                    }
                }
                catch (Exception e)
                {
                    // We need to keep processing the rest of the batch - capture this exception and continue.
                    exceptions.Add(e);
                }
            }

            if (exceptions.Count > 1)
                throw new AggregateException(exceptions);

            if (exceptions.Count == 1)
                throw exceptions.Single();
        }


        public static async Task<string> GetTwinId(DigitalTwinsClient client, string regId, ILogger log)
        {
            string query = $"SELECT * FROM DigitalTwins T WHERE T.HubRegistrationId = '{regId}'";
            AsyncPageable<string> twins = client.QueryAsync(query);
            await foreach (string twinJson in twins)
            {
                JObject twin = (JObject)JsonConvert.DeserializeObject(twinJson);
                string dtId = (string)twin["$dtId"];
                log.LogInformation($"Twin '{dtId}' found in DT");
                return dtId;
            }

            return null;
        }

        public static async Task DeleteRelationships(DigitalTwinsClient client, string dtId, ILogger log)
        {
            var relationshipIds = new List<string>();

            AsyncPageable<string> relationships = client.GetRelationshipsAsync(dtId);
            await foreach (var relationshipJson in relationships)
            {
                BasicRelationship relationship = System.Text.Json.JsonSerializer.Deserialize<BasicRelationship>(relationshipJson);
                relationshipIds.Add(relationship.Id);
            }

            foreach(var relationshipId in relationshipIds)
            {
                client.DeleteRelationship(dtId, relationshipId);
                log.LogInformation($"Twin '{dtId}' relationship '{relationshipId}' deleted in DT");
            }
        }
    }
}
```

### Configure your function

Next, you'll need to set environment variables in your function app from earlier, containing the reference to the Azure Digital Twins instance you've created.

You will need the following values from when you set up your instance. 
If you need to gather these values again, use the links below to the corresponding sections in the setup article for finding them in the [Azure portal](https://portal.azure.com).
* Azure Digital Twins instance **_host name_** ([find in portal](../articles/digital-twins/how-to-set-up-instance-portal.md#verify-success-and-collect-important-values))
* Azure AD app registration **_Application (client) ID_** ([find in portal](../articles/digital-twins/how-to-set-up-instance-portal.md#collect-important-values))

```azurecli-interactive
az functionapp config appsettings set --settings "AdtInstanceUrl=https://<Azure Digital Twins instance _host name_> -g <resource group> -n <your App Service (function app) name>"
```

```azurecli-interactive
az functionapp config appsettings set --settings "AdtAppId=<Application (client) ID> -g <resource group> -n <your App Service (function app) name>"
```

### Create an IoT Hub route for Lifecycle events

[todo]

### Validate

[todo]

## Cleanup

[todo]

## Next steps

The digital twins created for the devices are stored as a flat hierarchy in Azure Digital Twins, but they can be enriched with model information and a multi-level hierarchy for organization. To learn more about this concept, read: 

* [*Concept: Understand digital twins and their twin graph*](../digital-twins/concepts-twins-graph) 

You can write custom logic to automatically provide this information using the model and graph data already stored in Azure Digital Twins. To read more about managing, upgrading, and retrieving information from the twins graph, see the following references:

* [*How-to: Manage a digital twin*](./how-to-manage-twin.md)
* [*How-to: Query the twin graph*](./how-to-query-graph.md)