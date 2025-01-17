---
title: Microsoft Entra authentication for Application Insights
description: Learn how to enable Microsoft Entra authentication to ensure that only authenticated telemetry is ingested in your Application Insights resources.
ms.topic: conceptual
ms.date: 04/01/2024
ms.devlang: csharp
ms.reviewer: rijolly
---

# Microsoft Entra authentication for Application Insights

Application Insights now supports [Microsoft Entra authentication](../../active-directory/authentication/overview-authentication.md). By using Microsoft Entra ID, you can ensure that only authenticated telemetry is ingested in your Application Insights resources.

Using various authentication systems can be cumbersome and risky because it's difficult to manage credentials at scale. You can now choose to [opt out of local authentication](#disable-local-authentication) to ensure only telemetry exclusively authenticated by using [managed identities](../../active-directory/managed-identities-azure-resources/overview.md) and [Microsoft Entra ID](../../active-directory/fundamentals/active-directory-whatis.md) is ingested in your resource. This feature is a step to enhance the security and reliability of the telemetry used to make critical operational ([alerting](../alerts/alerts-overview.md#what-are-azure-monitor-alerts) and [autoscaling](../autoscale/autoscale-overview.md#overview-of-autoscale-in-azure)) and business decisions.

> [!NOTE]
> This document covers data ingestion into Application Insights using Microsoft Entra ID-based authentication. For information on querying data within Application Insights, see [Query Application Insights using Microsoft Entra authentication](./app-insights-azure-ad-api.md).

## Prerequisites

The following preliminary steps are required to enable Microsoft Entra authenticated ingestion. You need to:

- Be in the public cloud.
- Be familiar with:
  - [Managed identity](../../active-directory/managed-identities-azure-resources/overview.md).
  - [Service principal](../../active-directory/develop/howto-create-service-principal-portal.md).
  - [Assigning Azure roles](../../role-based-access-control/role-assignments-portal.yml).
- Granting access using [Azure built-in roles](../../role-based-access-control/built-in-roles.md) requires having an Owner role to the resource group.
- Understand the [unsupported scenarios](#unsupported-scenarios).

## Unsupported scenarios

The following Software Development Kits (SDKs) and features are unsupported for use with Microsoft Entra authenticated ingestion:

- [Application Insights Java 2.x SDK](deprecated-java-2x.md#monitor-dependencies-caught-exceptions-and-method-execution-times-in-java-web-apps).<br />
 Microsoft Entra authentication is only available for Application Insights Java Agent greater than or equal to 3.2.0.
- [ApplicationInsights JavaScript web SDK](javascript.md).
- [Application Insights OpenCensus Python SDK](/previous-versions/azure/azure-monitor/app/opencensus-python) with Python version 3.4 and 3.5.
- On-by-default [autoinstrumentation/codeless monitoring](codeless-overview.md) (for languages) for Azure App Service, Azure Virtual Machines/Azure Virtual Machine Scale Sets, and Azure Functions.
- [Profiler](profiler-overview.md).

<a name='configure-and-enable-azure-ad-based-authentication'></a>

## Configure and enable Microsoft Entra ID-based authentication

1. If you don't already have an identity, create one by using either a managed identity or a service principal.

    - We recommend using a managed identity:

        [Set up a managed identity for your Azure service](../../active-directory/managed-identities-azure-resources/services-support-managed-identities.md) (Virtual Machines or App Service).

    - We don't recommend using a service principal:

        For more information on how to create a Microsoft Entra application and service principal that can access resources, see [Create a service principal](../../active-directory/develop/howto-create-service-principal-portal.md).

1. Assign the required RBAC role to the Azure identity, service principal, or Azure user account.

    Follow the steps in [Assign Azure roles](../../role-based-access-control/role-assignments-portal.yml) to add the Monitoring Metrics Publisher role to the expected identity, service principal, or Azure user account by setting the target Application Insights resource as the role scope.

    > [!NOTE]
    > Although the Monitoring Metrics Publisher role says "metrics," it will publish all telemetry to the Application Insights resource.

1. Follow the configuration guidance in accordance with the language that follows.

### [.NET](#tab/net)

> [!NOTE]
> Support for Microsoft Entra ID in the Application Insights .NET SDK is included starting with [version 2.18-Beta3](https://www.nuget.org/packages/Microsoft.ApplicationInsights/2.18.0-beta3).

Application Insights .NET SDK supports the credential classes provided by [Azure Identity](https://github.com/Azure/azure-sdk-for-net/tree/master/sdk/identity/Azure.Identity#credential-classes).

- We recommend `DefaultAzureCredential` for local development.
- Ensure you're authenticated on Visual Studio with the expected Azure user account. For more information, see [Authenticate via Visual Studio](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/identity/Azure.Identity#authenticate-via-visual-studio).
- We recommend `ManagedIdentityCredential` for system-assigned and user-assigned managed identities.
  - For system-assigned, use the default constructor without parameters.
  - For user-assigned, provide the client ID to the constructor.

The following example shows how to manually create and configure `TelemetryConfiguration` by using .NET:

```csharp
TelemetryConfiguration.Active.ConnectionString = "InstrumentationKey=00000000-0000-0000-0000-000000000000;IngestionEndpoint=https://xxxx.applicationinsights.azure.com/";
var credential = new DefaultAzureCredential();
TelemetryConfiguration.Active.SetAzureTokenCredential(credential);
```

The following example shows how to configure `TelemetryConfiguration` by using .NET Core:

```csharp
services.Configure<TelemetryConfiguration>(config =>
{
       var credential = new DefaultAzureCredential();
       config.SetAzureTokenCredential(credential);
});
services.AddApplicationInsightsTelemetry(new ApplicationInsightsServiceOptions
{
    ConnectionString = "InstrumentationKey=00000000-0000-0000-0000-000000000000;IngestionEndpoint=https://xxxx.applicationinsights.azure.com/"
});
```

### [Node.js](#tab/nodejs)

> [!NOTE]
> Support for Microsoft Entra ID in the Application Insights Node.JS is included starting with [version 2.1.0-beta.1](https://www.npmjs.com/package/applicationinsights/v/2.1.0-beta.1).

Application Insights Node.JS supports the credential classes provided by [Azure Identity](https://github.com/Azure/azure-sdk-for-js/tree/master/sdk/identity/identity#credential-classes).

#### DefaultAzureCredential

```javascript
import appInsights from "applicationinsights";
import { DefaultAzureCredential } from "@azure/identity"; 
 
const credential = new DefaultAzureCredential();
appInsights.setup("InstrumentationKey=00000000-0000-0000-0000-000000000000;IngestionEndpoint=https://xxxx.applicationinsights.azure.com/").start();
appInsights.defaultClient.config.aadTokenCredential = credential;

```

### [Java](#tab/java)

> [!NOTE]
> Support for Microsoft Entra ID in the Application Insights Java agent is included starting with [Java 3.2.0-BETA](https://github.com/microsoft/ApplicationInsights-Java/releases/tag/3.2.0-BETA).

1. [Configure your application with the Java agent.](opentelemetry-enable.md?tabs=java#get-started)

    > [!IMPORTANT]
    > Use the full connection string, which includes `IngestionEndpoint`, when you configure your app with the Java agent. For example, use `InstrumentationKey=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX;IngestionEndpoint=https://XXXX.applicationinsights.azure.com/`.

1. Add the JSON configuration to the *ApplicationInsights.json* configuration file depending on the authentication you're using. We recommend using managed identities.

> [!NOTE]
> For more information about migrating from the `2.X` SDK to the `3.X` Java agent, see [Upgrading from Application Insights Java 2.x SDK](java-standalone-upgrade-from-2x.md).

#### System-assigned managed identity

The following example shows how to configure the Java agent to use system-assigned managed identity for authentication with Microsoft Entra ID.

```JSON
{ 
  "connectionString": "App Insights Connection String with IngestionEndpoint", 
  "authentication": { 
    "enabled": true, 
    "type": "SAMI" 
  } 
} 
```

#### User-assigned managed identity

The following example shows how to configure the Java agent to use user-assigned managed identity for authentication with Microsoft Entra ID.

```JSON
{ 
  "connectionString": "App Insights Connection String with IngestionEndpoint", 
  "authentication": { 
    "enabled": true, 
    "type": "UAMI", 
    "clientId":"<USER-ASSIGNED MANAGED IDENTITY CLIENT ID>" 
  } 
} 
```

:::image type="content" source="media/azure-ad-authentication/user-assigned-managed-identity.png" alt-text="Screenshot that shows user-assigned managed identity." lightbox="media/azure-ad-authentication/user-assigned-managed-identity.png":::

#### Environment variable configuration

The `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` environment variable lets Application Insights authenticate to Microsoft Entra ID and send telemetry.

- For system-assigned identity:

| App setting    | Value    |
| -------------- |--------- |
| APPLICATIONINSIGHTS_AUTHENTICATION_STRING         | `Authorization=AAD`    |

- For user-assigned identity:

| App setting   | Value    |
| ------------- | -------- |
| APPLICATIONINSIGHTS_AUTHENTICATION_STRING         | `Authorization=AAD;ClientId={Client id of the User-Assigned Identity}`    |

Set the `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` environment variable using this string.

**In Unix/Linux:**

```shell
export APPLICATIONINSIGHTS_AUTHENTICATION_STRING="Authorization=AAD"
```

**In Windows:**

```shell
set APPLICATIONINSIGHTS_AUTHENTICATION_STRING="Authorization=AAD"
```

After setting it, restart your application. It now sends telemetry to Application Insights using Microsoft Entra authentication.

### [Python](#tab/python)

> [!NOTE]
> Microsoft Entra authentication is only available for Python v2.7, v3.6, and v3.7. Support for Microsoft Entra ID in the Application Insights OpenCensus Python SDK
is included starting with beta version [opencensus-ext-azure 1.1b0](https://pypi.org/project/opencensus-ext-azure/1.1b0/).

> [!NOTE]
> [OpenCensus Python SDK is deprecated](https://opentelemetry.io/blog/2023/sunsetting-opencensus/), but Microsoft supports it until retirement on September 30, 2024. We now recommend the [OpenTelemetry-based Python offering](./opentelemetry-enable.md?tabs=python) and provide [migration guidance](./opentelemetry-python-opencensus-migrate.md?tabs=aspnetcore).

Construct the appropriate [credentials](/python/api/overview/azure/identity-readme#credentials) and pass them into the constructor of the Azure Monitor exporter. Make sure your connection string is set up with the instrumentation key and ingestion endpoint of your resource.

The `OpenCensus` Azure Monitor exporters support these authentication types. We recommend using managed identities in production environments.

#### System-assigned managed identity

```python
from azure.identity import ManagedIdentityCredential

from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer

credential = ManagedIdentityCredential()
tracer = Tracer(
    exporter=AzureExporter(credential=credential, connection_string="InstrumentationKey=<your-instrumentation-key>;IngestionEndpoint=<your-ingestion-endpoint>"),
    sampler=ProbabilitySampler(1.0)
)
...

```

#### User-assigned managed identity

```python
from azure.identity import ManagedIdentityCredential

from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer

credential = ManagedIdentityCredential(client_id="<client-id>")
tracer = Tracer(
    exporter=AzureExporter(credential=credential, connection_string="InstrumentationKey=<your-instrumentation-key>;IngestionEndpoint=<your-ingestion-endpoint>"),
    sampler=ProbabilitySampler(1.0)
)
...

```

---

## Disable local authentication

After the Microsoft Entra authentication is enabled, you can choose to disable local authentication. This configuration allows you to ingest telemetry authenticated exclusively by Microsoft Entra ID and affects data access (for example, through API keys).

You can disable local authentication by using the Azure portal or Azure Policy or programmatically.

### Azure portal

1. From your Application Insights resource, select **Properties** under the **Configure** heading in the menu on the left. Select **Enabled (click to change)** if the local authentication is enabled.

   :::image type="content" source="./media/azure-ad-authentication/enabled.png" alt-text="Screenshot that shows Properties under the Configure section and the Enabled (select to change) local authentication button.":::

1. Select **Disabled** and apply changes.

   :::image type="content" source="./media/azure-ad-authentication/disable.png" alt-text="Screenshot that shows local authentication with the Enabled/Disabled button.":::

1. After disabling local authentication on your resource, you'll see the corresponding information in the **Overview** pane.

   :::image type="content" source="./media/azure-ad-authentication/overview.png" alt-text="Screenshot that shows the Overview tab with the Disabled (select to change) local authentication button.":::

### Azure Policy

Azure Policy for `DisableLocalAuth` denies users the ability to create a new Application Insights resource without this property set to `true`. The policy name is `Application Insights components should block non-AAD auth ingestion`.

To apply this policy definition to your subscription, [create a new policy assignment and assign the policy](../../governance/policy/assign-policy-portal.md).

The following example shows the policy template definition:

```JSON
{
    "properties": {
        "displayName": "Application Insights components should block non-AAD auth ingestion",
        "policyType": "BuiltIn",
        "mode": "Indexed",
        "description": "Improve Application Insights security by disabling log ingestion that are not AAD-based.",
        "metadata": {
            "version": "1.0.0",
            "category": "Monitoring"
        },
        "parameters": {
            "effect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect",
                    "description": "The effect determines what happens when the policy rule is evaluated to match"
                },
                "allowedValues": [
                    "audit",
                    "deny",
                    "disabled"
                ],
                "defaultValue": "audit"
            }
        },
        "policyRule": {
            "if": {
                "allOf": [
                    {
                        "field": "type",
                        "equals": "Microsoft.Insights/components"
                    },
                    {
                        "field": "Microsoft.Insights/components/DisableLocalAuth",
                        "notEquals": "true"                        
                    }
                ]
            },
            "then": {
                "effect": "[parameters('effect')]"
            }
        }
    }
}
```

### Programmatic enablement

The property `DisableLocalAuth` is used to disable any local authentication on your Application Insights resource. When this property is set to `true`, it enforces that Microsoft Entra authentication must be used for all access.

The following example shows the Azure Resource Manager template you can use to create a workspace-based Application Insights resource with `LocalAuth` disabled.

```JSON
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "string"
        },
        "type": {
            "type": "string"
        },
        "regionId": {
            "type": "string"
        },
        "tagsArray": {
            "type": "object"
        },
        "requestSource": {
            "type": "string"
        },
        "workspaceResourceId": {
            "type": "string"
        },
        "disableLocalAuth": {
            "type": "bool"
        }
     
    },
    "resources": [
        {
        "name": "[parameters('name')]",
        "type": "microsoft.insights/components",
        "location": "[parameters('regionId')]",
        "tags": "[parameters('tagsArray')]",
        "apiVersion": "2020-02-02-preview",
        "dependsOn": [],
        "properties": {
            "Application_Type": "[parameters('type')]",
            "Flow_Type": "Redfield",
            "Request_Source": "[parameters('requestSource')]",
            "WorkspaceResourceId": "[parameters('workspaceResourceId')]",
            "DisableLocalAuth": "[parameters('disableLocalAuth')]"
            }
    }
 ]
}

```

### Token audience

When developing a custom client to obtain an access token from Microsoft Entra ID for submitting telemetry to Application Insights, refer to the following table to determine the appropriate audience string for your particular host environment.

| Azure cloud version | Token audience value |
| --- | --- |
| Azure public cloud | `https://monitor.azure.com` |
| Microsoft Azure operated by 21Vianet cloud | `https://monitor.azure.cn` |
| Azure US Government cloud | `https://monitor.azure.us` |

If you're using sovereign clouds, you can find the audience information in the connection string as well. The connection string follows this structure:

*InstrumentationKey={profile.InstrumentationKey};IngestionEndpoint={ingestionEndpoint};LiveEndpoint={liveDiagnosticsEndpoint};AADAudience={aadAudience}*

The audience parameter, AADAudience, can vary depending on your specific environment.

## Troubleshooting

This section provides distinct troubleshooting scenarios and steps that you can take to resolve an issue before you raise a support ticket.

### Ingestion HTTP errors

The ingestion service returns specific errors, regardless of the SDK language. Network traffic can be collected by using a tool such as Fiddler. You should filter traffic to the ingestion endpoint set in the connection string.

#### HTTP/1.1 400 Authentication not supported

This error shows the resource is set for Microsoft Entra-only. You need to correctly configure the SDK because it's sending to the wrong API.

> [!NOTE]
> "v2/track" doesn't support Microsoft Entra ID. When the SDK is correctly configured, telemetry will be sent to "v2.1/track".

Next, you should review the SDK configuration.

#### HTTP/1.1 401 Authorization required

This error indicates that the SDK is correctly configured but it's unable to acquire a valid token. This error might indicate an issue with Microsoft Entra ID.

Next, you should identify exceptions in the SDK logs or network errors from Azure Identity.

#### HTTP/1.1 403 Unauthorized

This error means the SDK uses credentials without permission for the Application Insights resource or subscription.

First, check the Application Insights resource's access control. You must configure the SDK with credentials that have the Monitoring Metrics Publisher role.

### Language-specific troubleshooting

### [.NET](#tab/net)

#### Event source

The Application Insights .NET SDK emits error logs by using the event source. To learn more about collecting event source logs, see [Troubleshooting no data - collect logs with PerfView](asp-net-troubleshoot-no-data.md#PerfView).

If the SDK fails to get a token, the exception message is logged as
`Failed to get AAD Token. Error message:`.

### [Node.js](#tab/nodejs)

Internal logs could be turned on by using the following setup. After they're enabled, error logs will be shown in the console, including any error related to Microsoft Entra integration. Examples include failure to generate the token when the wrong credentials are supplied or errors when the ingestion endpoint fails to authenticate by using the provided credentials.

```javascript
let appInsights = require("applicationinsights");
appInsights.setup("InstrumentationKey=00000000-0000-0000-0000-000000000000;IngestionEndpoint=https://xxxx.applicationinsights.azure.com/").setInternalLogging(true, true);
```

### [Java](#tab/java)

#### HTTP traffic

You can inspect network traffic by using a tool like Fiddler. To enable the traffic to tunnel through Fiddler, either add the following proxy settings in the configuration file:

```JSON
"proxy": {
"host": "localhost",
"port": 8888
}
```

Or add the following Java Virtual Machine (JVM) args while running your application: `-Djava.net.useSystemProxies=true -Dhttps.proxyHost=localhost -Dhttps.proxyPort=8888`

If Microsoft Entra ID is enabled in the agent, outbound traffic includes the HTTP header `Authorization`.

#### 401 Unauthorized

If you see the message, `WARN c.m.a.TelemetryChannel - Failed to send telemetry with status code: 401, please check your credentials` in the log, it means the agent couldn't send telemetry. You likely didn't enable Microsoft Entra authentication on the agent, while your Application Insights resource has `DisableLocalAuth: true`. Ensure you pass a valid credential with access permission to your Application Insights resource.

If you're using Fiddler, you might see the response header `HTTP/1.1 401 Unauthorized - please provide the valid authorization token`.

#### CredentialUnavailableException

If you see the exception, `com.azure.identity.CredentialUnavailableException: ManagedIdentityCredential authentication unavailable. Connection to IMDS endpoint cannot be established` in the log file, it means the agent failed to acquire the access token. The likely cause is an invalid client ID in your User-Assigned Managed Identity configuration.

#### Failed to send telemetry

If you see the message, `WARN c.m.a.TelemetryChannel - Failed to send telemetry with status code: 403, please check your credentials` in the log, it means the agent couldn't send telemetry. The likely reason is that the credentials used don't allow telemetry ingestion.

Using Fiddler, you might notice the response `HTTP/1.1 403 Forbidden - provided credentials do not grant the access to ingest the telemetry into the component`.

The issue could be due to:

- Creating the resource with a system-assigned managed identity or associating a user-assigned identity without adding the Monitoring Metrics Publisher role to it.
- Using the correct credentials for access tokens but linking them to the wrong Application Insights resource. Ensure your resource (virtual machine or app service) or user-assigned identity has Monitoring Metrics Publisher roles in your Application Insights resource.

#### Invalid Client ID

If the exception, `com.microsoft.aad.msal4j.MsalServiceException: Application with identifier <CLIENT_ID> was not found in the directory` in the log, it means the agent failed to get the access token. This exception likely happens because the client ID in your client secret configuration is invalid or incorrect.

This issue occurs if the administrator doesn't install the application or no tenant user consents to it. It also happens if you send your authentication request to the wrong tenant.

### [Python](#tab/python)

#### Error starts with "credential error" (with no status code)

Something is incorrect about the credential you're using and the client isn't able to obtain a token for authorization. It's because the required data is lacking for the state. An example would be passing in a system `ManagedIdentityCredential` but the resource isn't configured to use system-managed identity.

#### Error starts with "authentication error" (with no status code)

The client failed to authenticate with the given credential. This error usually occurs when the credential used doesn't have the correct role assignments.

#### I'm getting a status code 400 in my error logs

You're probably missing a credential or your credential is set to `None`, but your Application Insights resource is configured with `DisableLocalAuth: true`. Make sure you're passing in a valid credential and that it has permission to access your Application Insights resource.

#### I'm getting a status code 403 in my error logs

This error usually occurs when the provided credentials don't grant access to ingest telemetry for the Application Insights resource. Make sure your Application Insights resource has the correct role assignments.

---

## Next steps

- [Monitor your telemetry in the portal](overview-dashboard.md)
- [Diagnose with Live Metrics Stream](live-stream.md)
- [Query Application Insights using Microsoft Entra authentication](./app-insights-azure-ad-api.md)
