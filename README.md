# Use Proxy Provider with Teams Toolkit

## Introduction
When you use the proxy provider, you can use your backend authentication (such as Auth2.0 On-Behalf-Of flow) to power the Microsoft Graph Toolkit by routing all calls to Microsoft Graph through your own backend.

Your backend service must expose an API that will be called for every call to Microsoft Graph. For example, when a component attempts to get a resource, the ProxyProvider will instead call your base API and append that resource.


## User scenario
The customer is developing a tab and using "People Picker" and "Person" components from "graph toolkit". And they use teamsfx as the auth provider to enable the authentication in graph toolkit.
It works well in desktop in the company. However, it cannot work on mobile. Because the tenant turned on the "conditional access" and mobile device cannot exchange token through the auth code flow in frontend which is wrapped by default in our teamsfx sdk.

In this scenario, we can use proxy provider for graph toolkit

## Try the Sample
1. Click F5 start the Tab project

2. When debug browser popup, add `proxy-provider-node/.env` file as below (you can find these values from `.fx/states/state.local.json`)

  ```
    GRAPH_HOST='https://graph.microsoft.com'
    AUTH_PASS_THROUGH=false
    PROXY_APP_ID='xxx'
    PROXY_APP_TENANT_ID='xxx'
    PROXY_APP_SECRET='xxx'
  ```

3. Start node proxy server

  ```bash
    npm install
    npm run build
    npm run start
  ```

4. Go to debug browser and try the teams app


## Related code

Use proxy provider:
```ts
const { loading, error, data, reload } = useGraph(
    async (graph, teamsfx, scope) => {
      ...
      const provider = new ProxyProvider('http://localhost:8000/apiproxy', async() => {
        // This code executes for each call to the proxy to
        // get any headers that it should add to the request.
        const ssoToken = await teamsfx.getCredential().getToken("");
        return { Authorization: `Bearer ${ssoToken?.token}` };
      });

      Providers.globalProvider = provider;
      Providers.globalProvider.setState(ProviderState.SignedIn);

      ...
    },
    { scope: ["User.Read"], teamsfx: teamsfx }
  );
```


## Use proxy provider with APIM
We can use APIM as backend instead of write your own proxy server. APIM support policy to change the behavior of the API through, and it can be used to exchange token.
Below is an example to exchange token with client credential flow.

```xml
<!-- The policy defined in this file provides an example of using OAuth2 for authorization between the gateway and a backend. -->
<!-- It shows how to obtain an access token from Azure AD and forward it to the backend. -->

<!-- Send request to Azure AD to obtain a bearer token -->
<!-- Parameters: authorizationServer - format https://login.windows.net/TENANT-GUID/oauth2/token -->
<!-- Parameters: scope - a URI encoded scope value -->
<!-- Parameters: clientId - an id obtained during app registration -->
<!-- Parameters: clientSecret - a URL encoded secret, obtained during app registration -->

<!-- Copy the following snippet into the inbound section. -->

<policies>
  <inbound>
    <base />
      <send-request ignore-error="true" timeout="20" response-variable-name="bearerToken" mode="new">
        <set-url>{{authorizationServer}}</set-url>
        <set-method>POST</set-method>
        <set-header name="Content-Type" exists-action="override">
          <value>application/x-www-form-urlencoded</value>
        </set-header>
        <set-body>
          @{
              return "client_id={{clientId}}&scope={{scope}}&client_secret={{clientSecret}}&grant_type=client_credentials";

              // For Azure AD v1, try return statement below
              // return "client_id={{clientId}}&resource={{scope}}&client_secret={{clientSecret}}&grant_type=client_credentials";
          }
        </set-body>
      </send-request>

      <set-header name="Authorization" exists-action="override">
        <value>
          @("Bearer " + (String)((IResponse)context.Variables["bearerToken"]).Body.As<JObject>()["access_token"])
      </value>
      </set-header>

      <!--  Don't expose APIM subscription key to the backend. -->
      <set-header exists-action="delete" name="Ocp-Apim-Subscription-Key"/>
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
```
