# DevOpsAcademyChat
This is a test project for DevOps Academy


In Order to build the project working in Azure you need to follow the following several steps:

# 1. Create Azure Function App 3.0 
Basic settings:
- Publish: Code
- Runtime stack: .Net Core
- Region: [best for you]
Hosting settings:
- Operating Systems: Windows
- Payment Plan: Consumtion
Monitoring settings:
- Application Insights: No

# 2. Create Azure SignalR Service
Basic settings:
- Pricing tier: Free
- Service Mode: Serverless

Remark: you will need this service' endpoint connection string.

# 3. Create Negotiation Trigger in Azure Function App
Open the created Azure Function and add new trigger:
- find trigger type: 'SignalR negotiate HTTP trigger'
- Name: negotiate
- RouterTemplate: negotiate
- Hubname: degault
- SignalR Service connection: <new> and use SignalR endpoint connection string

function code
```C#
#r "Microsoft.Azure.WebJobs.Extensions.SignalRService"
using Microsoft.Azure.WebJobs.Extensions.SignalRService;

public static SignalRConnectionInfo Run(HttpRequest req, SignalRConnectionInfo connectionInfo)
{
    return connectionInfo;
}
```

function.json
```JSON
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "post"
      ],
      "name": "req",
      "route": "negotiate"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    },
    {
      "type": "signalRConnectionInfo",
      "name": "connectionInfo",
      "hubName": "default",
      "connectionStringSetting": "SIGNALR_KEY",
      "direction": "in"
    }
  ],
  "disabled": false
}
```

# 4. Create Message Trigger in Azure Function App
Open the created Azure Function and add new trigger:
- find trigger type: 'SignalR negotiate HTTP trigger'
- Name: messages
- RouterTemplate: messages
- Hubname: degault
- SignalR Service connection: reuse the previous connection name
Integration changes
- change the input type from `signalRConnectionInfo` to `signalR` (best do it in function.json)

function code
```C#
#r "Microsoft.Azure.WebJobs.Extensions.SignalRService"
using Microsoft.Azure.WebJobs.Extensions.SignalRService;

public static Task Run(object message, IAsyncCollector<SignalRMessage> signalRMessages)
{
    return signalRMessages.AddAsync(
                new SignalRMessage
                {
                    Target = "newMessage",
                    Arguments = new[] { message }
                });
}
```

function.json
```JSON
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "post"
      ],
      "name": "message",
      "route": "messages"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    },
     {
      "type": "signalR",
      "name": "signalRMessages",
      "hubName": "default",
      "connectionStringSetting": "SIGNALR_KEY",
      "direction": "in"
    }
  ],
  "disabled": false
}
```

# 5. Crate Azure Storage Account
Base settings:
- Performance: Standard
- Account kind: Storage V2
- Replication: LRS
- Access tier: Hot
Networking settings:
- Connectivity method: Public Endpoint

After creation enable 'Static website' service

# 6. Prepare files - index.html & index-anonymous.html
Change this URL to the address of your Azure function application.
```JavaScript
   const apiBaseUrl = 'https://[youfuncapp].azurewebsites.net';
```

# 7. Upload files
Upload `index.html` & & `index-anonymous.html` to `$web` container in your storage account

Check: try `https://[youfuncapp].azurewebsites.net/index-anonymous.html` to access chat pache without authentication 

# 8. Azure Active Direcctory B2C
Configuration of AD B2C is not included in this document

# 8. Active Directory B2C application registration
Go to your Azure B2C tenant and register the url of the azure function application:
- Supported account types: Accounts in any organizational directory or any identity provider. For authenticating users with Azure AD B2C
- Redirect URI: `https://[youfuncapp].azurewebsites.net/.auth/login/aad/callback`
- Permissions
  - Grant admin consent to openid and offline_access permissions: Yes
Remark: get the 'ClientId'

# 8. Enable Authentication
Go to your Azure Function App / Platform reatures and open 'Authentication / Authorization' section
- App Service Authentication: ON
- Action to take when request is not authenticated: Log in with Azure Active Directory
- Authentication Providers:
  - Azure Active Directory Settings
    - Management mode: Advanced
    - Client ID: `[your application registration id from B2C]`
    - Issuer Url: `https://[youb2cname].b2clogin.com/[youb2cname].onmicrosoft.com/v2.0/.well-known/openid-configuration?p=[youb2c]]`
- Advanced Settings
  - Token Store: On
  - Allowed External Redirect URLs: `[your static web app url in storage account]`