# Use Proxy Provider with Teams Toolkit

## Introduction
When you use the proxy provider, you can use your backend authentication (such as Auth2.0 On-Behalf-Of flow) to power the Microsoft Graph Toolkit by routing all calls to Microsoft Graph through your own backend.

Your backend service must expose an API that will be called for every call to Microsoft Graph. For example, when a component attempts to get a resource, the ProxyProvider will instead call your base API and append that resource.

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
