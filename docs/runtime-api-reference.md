> Note: Azure Token Store is in Private Preview, and can be accessed by contacting our team, and completing the onboarding process under NDA. Private Preview features like Token Store are primarily meant to gather feedback, and should not be used in production. Support may be discontinued in the future.

# What are runtime APIs?

The runtime APIs are available through a dedicated web address for each token store. They provide token operations like get-valid-token, in addition to supporting most of the resource management actions. Runtime APIs are meant for direct programmatic access and are better optimized for performance. The service host is typically available at `https://[token-store-name].tokenstore.azure.net`.

# Calling the APIs with Rest Client extension

## Pre-requisites

1. [Visual Studio Code](https://code.visualstudio.com/) installed on your machine.
1. [HTTP REST Client extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) installed in Visual Studio Code
1. [Git command line](https://git-scm.com/)

## Getting started with `.http` files

1. Using command line to navigate to an empty folder and clone the repo by running the command `git clone https://github.com/Azure/azure-tokens.git`.
1. Run `code azure-tokens` to open the cloned folder in Visual Studio Code.
1. Open `/docs/runtime-api-reference.http` in Visual Studio Code to get started.
1. Update the following variables to reflect you own Azure environment.
    1. **`@subscriptionId`** should be set to the your Azure subscription ID.
    1. **`@resourceGroup`** should be set to the a resource you have contributor access to.
    1. **`@storeName`** should be set to the store name you are interacting with.
    1. Replace `[your-domain.com]` with your domainname. For example if your personal user Id is *me@mycompany.com*, the string [your-domain.com] should be replaced with `mycompany.com` everywhere in the `.http` document.
1. Under the **Read token** section, click on the `Send request` link above `GET {{runtimeBaseUrl}}/services/{{serviceName}}/tokens/{{tokenName}}`. The *REST Client* extension will walk you through logging into Azure Resource Manager and resolve the value for `{{$aadToken [your-domain.com] aud:https://[token-store-name].tokenstore.azure.net}}`:
    1. Click on **Sign in** when a VS Code prompts you to sign in. The authorization code will get copied to your clipboard.
    1. Proceed through browser sign in by pasting in the authorization code and proceeding with usual AAD login.
    1. Upon completion of sign in on browser, return to VS code and click **Done**.
1. The **Response** window will appear for your HTTP request in VS code containing response code, headers and a body. If working correctly, it will return status code `200 OK` and response body containing a JSON list of the resource group's token stores.

# Security

All calls to the token store ARM APIs must bcontain a JWT security token `Authorization` header, where the JWT token in turn contains `"aud": "https://[token-store-name].tokenstore.azure.net"`. The principal of this token must also have appropriate access through the Token Store's access policy security model. See [Access Policy](/docs/access-policy-runtime-security.md) for more details on how this security model works.

One way to obtain a security token for this purpose is by using the code below, which in turn depends on the `AzureServiceTokenProvider` library.

```cs
var azureServiceTokenProvider = new AzureServiceTokenProvider("https://[token-store-name].tokenstore.azure.net");
string securityToken = await azureServiceTokenProvider.GetAccessTokenAsync(TokenStoreResource);
```

In the accompanying [.http file](/docs/runtime-api-reference.http), this header is automatically generated and sent as part of the Authorization header using the expression below.

```http
Authorization: {{$aadToken [your-domain.com] aud:https://[token-store-name].tokenstore.azure.net}}
```

# Token runtime operations

## Create or update token

A token can be created by requesting a `PUT` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/[token-name]` with the payload below. The result should be `201 Created`.

```json
{
  "properties": {
    "displayName": "Test Dropbox token"
  }
}
```

Try this operation out in VS Code under **Create or update token** section of the [.http file](/docs/runtime-api-reference.http).

## Read token

A token can be read by requesting a `GET` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/[token-name]`. The resulting payload will be similar to below:

```json
{
    "name": "testtoken",
    "displayName": "testToken" , 
    "parameterValues": {},
    "tokenUri": "https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/testtoken",
    "loginUri": "https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/testtoken/login",
    "value": null,
    "status": {
        "state": "Error",
        "error": {
        "code": "Unauthenticated",
        "message": "This connection is not authenticated."
        }
    },
    "location": "westcentralus",  
}
```

Try this operation out in VS Code under **Read token** section of the [.http file](/docs/runtime-api-reference.http).

## Save token (after post-login redirect)

Fully secure integration of Token Store with your web application requires implementing the post-login redirect pattern. Please refer to the [Phishing attack vulnerability](phishing-attack-vulnerability.md) topic for the reasons why this is required. The said pattern includes a last step of saving the token by requesting a `POST` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/[token-name]/save` with the payload below. The result should be `200 OK`.

```json
{
  "code": "[code-acquired-from-redirect]"
}
```

The response in turn will contain the following data:

```json
{
  "Parameters": {},
  "Value": {
    "AccessToken": "[valid-access-token]"
  },
  "ExpiresIn": "[access-token-expiry-date]"
}
```

Try this operation out in VS Code under **Save token** section of the [.http file](/docs/runtime-api-reference.http).

## Delete token

A token can be read by requesting a `DELETE` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/[token-name]`with no payload. The result should be `200 OK`.

Try this operation out in VS Code under **Delete token** section of the [.http file](/docs/runtime-api-reference.http).

## List tokens

Tokens can be listed by requesting a `GET` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens`. The result should be `200 OK` with a payload similar to below. 

```json
{
  "value": [{
    "name": "testtoken1",
    "displayName": "Test token 1",
    "parameterValues": {},
    "tokenUri": "https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/testtoken1",
    "loginUri": "https://[token-store-name].tokenstore.azure.net/services/[service-name]/tokens/testtoken1/login",
    "value": null,
    "status": {
    "state": "Error",
    "error": {
        "code": "Unauthenticated",
        "message": "This connection is not authenticated."
    }
    }, 
    "location": "westcentralus" 
  },
  ...
  ]
}
```

Try this operation out in VS Code under **List tokens** section of the [.http file](/docs/runtime-api-reference.http).

# Service runtime operations

## Create or update service

A service can be created by requesting a `PUT` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]` with the payload below. The result should be `201 Created`.

```json
{
    "authentication": {
      "managedIdentityProvider": {
        "name": "dropbox"
      },
      "parameters": {
        "clientid": "[dropbox-client-id]",
        "clientsecret": "[dropbox-client-secret]"
      }
    },
    "displayName": "Test Dropbox service"
}
```

Try this operation out in VS Code under **Create or update service** section of the [.http file](/docs/runtime-api-reference.http).

## Read service

A service can be read by requesting a `GET` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]`. The resulting payload will be similar to below. 

```json
{
    "tokenParameters": {},
    "authentication": {
      "managedIdentityProvider": {
        "name": "dropbox"
      }
    },
    "displayName": "Test Dropbox service"  
}
```

Try this operation out in VS Code under **Read service** section of the [.http file](/docs/runtime-api-reference.http).

## Delete service

A service can be read by requesting a `DELETE` operation against `https://[token-store-name].tokenstore.azure.net/services/[service-name]`with no payload. The result should be `200 OK`.

Try this operation out in VS Code under **Delete service** section of the [.http file](/docs/runtime-api-reference.http).

## List services

Services can be listed by requesting a `GET` operation against `https://[token-store-name].tokenstore.azure.net/services`. The result should be `200 OK` with a payload similar to below.

```json
{
  "value": [{
      "tokenParameters": {},
      "authentication": {
        "managedIdentityProvider": {
          "name": "dropbox"
        }
      },
      "displayName": "testService1"
  },
  ...
  ]
}
```

Try this operation out in VS Code under **List services** section of the [.http file](/docs/runtime-api-reference.http).
