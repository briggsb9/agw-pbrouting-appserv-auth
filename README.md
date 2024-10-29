# Using Authentication with Azure App Service Behind Application Gateway with Path-Based Routing

Azure Application Gateway (App Gateway) provides the ability to route traffic to multiple backends using path-based routing. However, when integrating authentication services, such as Azure App Service authentication (otherwise known as easy auth), complexities arise. This guidance discusses challenges and considerations when implementing authentication in App Service behind an App Gateway with path-based routing.

---

## Contents

1. [Understanding the Challenge with Path-Based Routing and Authentication](#understanding-the-challenge-with-path-based-routing-and-authentication)
2. [Recommended Approach: Subdomain Routing](#recommended-approach-subdomain-routing)
3. [Path-Based Routing Options](#path-based-routing-options)
   - [Option 1: Easy Auth with Host Header Override](#option-1-easy-auth-with-host-header-override)
   - [Option 2: Code-Based Authentication with Host Header Override](#option-2-code-based-authentication-with-host-header-override)
4. [Pros and Cons of Each Approach](#pros-and-cons-of-each-approach)
5. [Key Considerations and Best Practices](#key-considerations-and-best-practices)

---

## Understanding the Challenge with Path-Based Routing and Authentication

Path-based routing directs traffic based on URL paths, which can create conflicts with authentication workflows that expect specific hostnames or domains.

Challenges include:
- **Callback URLs** must match the configuration in the Azure App Registration and authentication settings.
- **Host Headers** might be overridden or mismatched due to App Gateway settings, leading to unexpected redirects.
- **Path-based routing rules** require careful configuration to avoid conflicts, particularly with wildcard entries or overlapping path patterns.

---

## Ideal Approach: Subdomain Routing

Using subdomains instead of path-based routing simplifies authentication. With subdomains, each application behind the App Gateway can maintain unique authentication configurations without conflicting callback paths. For instance, `https://app1.example.com` and `https://app2.example.com` can independently handle Easy Auth or other forms of authentication without risking path-based conflicts.

### Benefits of Subdomain Routing:
- **Simplified Configuration**: Each app can have independent callback URLs, reducing overlap and the need for custom configuration.
- **Reduced Risks**: Avoids issues with overlapping paths, host header overrides, and wildcard routing conflicts.
- **More Reliable Auth Flows**: Supports a more stable integration with Easy Auth or other identity providers.

However, if subdomain routing is not feasible, path-based routing can be used with the following configurations.

---

## Path-Based Routing Options

### Option 1: Easy Auth with Host Header Override

Using Azure App Service’s Easy Auth with path-based routing requires configuring the host headers and authentication settings to ensure that requests respect the path and route configurations.

Note: The steps below assume you have an existing Azure App Service with [easy auth](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization) enabled and an Azure Application Gateway set up with [path-based routing](https://learn.microsoft.com/en-us/azure/application-gateway/create-url-route-portal) rules.

#### Additional Steps:
1. **App Gateway Configuration**: Set up the backend HTTP setting in App Gateway to pick the hostname from the backend target. This uses the default App Service hostname to avoid domain conflicts. 

>[!WARNING]
> Verify the impact of using the default hostname by consulting [Azure’s host name preservation guide](https://learn.microsoft.com/en-us/azure/architecture/best-practices/host-name-preservation).

2. **App Registration Callback URL**: Ensure the app registration includes the correct callback URL, including the path (e.g., `https://yourdomain.com/yourpath/.auth/login/aad/callback`).

3. **Easy Auth Configuration (auth.json)**:
    - When using App Service Easy Auth behind Application Gateway, authentication redirects default to the app's Azure domain, often causing errors. To fix this, configure Easy Auth to read the X-Original-Host header from Application Gateway using file-based configuration as described in [Azure’s documentation](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-file-based#enabling-file-based-configuration).  

    - Update your `auth.json` to define the required HTTP settings:
        ```json
        "httpSettings": {
        "requireHttps": true,
        "routes": {
            "apiPrefix": "/YOUR_APP_PATH/.auth"
        },
        "forwardProxy": {
            "convention": "Custom",
            "customHostHeaderName": "X-Original-Host"
        }
        }
        ```
    - If issues persist, refer to the [configuration file reference](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-file-based#configuration-file-reference) and ensure you have the correct configuration for your scenario. For example, adding a value for  `"allowedExternalRedirectUrls"` to include `https://YOURDOMAIN.com/YOURPATH`.

>[!NOTE]
> One aspect not well documented is the value of "apiPrefix". It should match your app path to ensure Easy Auth respects the path-based routing. I've included this above for clarity.

#### Pros:
- **Minimal Code Changes**: Allows for quick integration with Easy Auth.
- **Host Header Management**: Ensures original host headers are respected.

#### Cons:
- **Complex Configuration**: Requires careful management of `auth.json` and App Gateway settings.
- **Limited Flexibility**: Easy Auth’s configurations may not cover all custom scenarios, especially with path-based routing.

---

### Option 2: Code-Based Authentication with Host Header Override

For scenarios requiring more flexibility, implementing custom authentication within the app code can be beneficial. This allows for handling authentication requests programmatically, which can help with maintaining control over redirect paths and headers. With this approach the user can be presented with a custom login page before accessing the app.

Note: The steps below assume you have an existing Azure App Service and an Azure Application Gateway set up with [path-based routing](https://learn.microsoft.com/en-us/azure/application-gateway/create-url-route-portal) rules.

#### Steps:
1. **Set up Authentication in Code**: Configure your app to authenticate users using a code-based approach with Microsoft Entra ID. Refer to the [Quickstart for Python Flask web app](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-v2-python-webapp) to set up authentication.

2. **Path-Based Routing Adjustments**:
   - Update the app routes in app.py to ensure your app responds to the path set in App Gateway. Each route must be configured to accept requests at the specified path (e.g., `@app.route("/YOURPATH")`).
   - Define the `REDIRECT_PATH` in `app_config.py` with the format `/YOURPATH/getAToken`.

#### Pros:
- **Full Control**: Customizable to fit specific routing and authentication requirements.
- **Greater Flexibility**: Allows complex configurations that may not be possible with Easy Auth.

#### Cons:
- **More Development Effort**: Requires additional coding and testing.
- **Manual Host Header Management**: You’ll need to handle headers, paths, and redirects carefully to ensure compatibility with App Gateway.

---

## Pros and Cons of Each Approach

| Approach                          | Pros                                 | Cons                                  |
|-----------------------------------|--------------------------------------|---------------------------------------|
| **Easy Auth with Host Override**  | Minimal code changes, preserves headers | Complex configuration, limited flexibility |
| **Code-Based Authentication**     | Full control, flexible               | Requires coding, manual configuration |

---

## Key Considerations and Best Practices

1. **Host Headers and Redirect URLs**: Ensure that host headers match expected values to prevent redirect mismatches.
2. **Callback URL Consistency**: Confirm that callback URLs are consistently configured in both App Service and Azure AD to match the path-based routing setup.
3. **Avoid Overlapping Paths**: Be cautious with wildcard entries in routing (e.g., `/app*`), which may unintentionally capture routes for other applications.
4. **Health Probes**: When using App Gateway, configure health probes to target the correct path (e.g., `/YOUR_APP_PATH/`) and verify that App Service responds correctly.

---

By following these guidelines, you can implement reliable and secure authentication for Azure App Service applications behind an Application Gateway using path-based routing. Each approach offers trade-offs, so select the one that best aligns with your architecture and operational needs.
