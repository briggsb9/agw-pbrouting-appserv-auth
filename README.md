# Understanding Azure App Service Authentication Challenges with Path-Based Routing

Azure Application Gateway supports path-based routing, allowing traffic to be directed based on the URL path of a request. Whilst App Service is compatible with this approach, managing multiple apps or mixed authentication requirements can introduce significant complexity. This article explores the challenges and key considerations when implementing path-based routing with App Service authentication.

---

## Contents

1. [Host-Based vs Path-Based Routing](#Host-Based-vs-Path-Based-Routing)
2. [App Service Authentication Options with Path-Based Routing ](#app-service-authentication-options-with-path-based-routing)
   - [Option 1: Easy Auth with Host Header Override](#option-1-easy-auth-with-host-header-override)
   - [Option 2: Code-Based Authentication with Host Header Override](#option-2-code-based-authentication-with-host-header-override)
   - [Options Summary](#options-summary)
3. [Conclusion](#general-tips)

---

## Host Based vs Path Based Routing

Firstly, it's important to understand that path-based routing is not the only option.

When building web applications, ensuring users can access the right content efficiently is crucial. One of the most important decisions in structuring web traffic is choosing between host-based and path-based routing. Both approaches can be used to determine how incoming requests are directed to different parts of your application(s).

So why does this matter when configuring authentication? In modern authentication flows, the identity provider redirects users after successful login to what is known as a callback URL. This URL needs to be correctly configured both in your app’s settings and in the identity provider’s configuration to ensure the authentication flow completes properly. A mismatch or misconfiguration of the callback URL can break the authentication process, causing redirect errors or security issues.

With that in mind, lets looks at each approach in more detail.

### Host-Based Routing

Host-based routing is often the preferred choice when each app has distinct requirements or independent functionality. In this approach, each application is assigned a separate hostname or subdomain (e.g., https://app1.domain.com and https://app2.domain.com), meaning each application behind the App Gateway can maintain unique authentication configurations without conflicting callback paths.

### Path-Based Routing

Path-based routing, however, is often used when the apps or services are related and need to share a common base URL. Examples include, sites with shared authentication or endpoints that are part of the same application. In these cases, path-based routing may be preferable, but more nuanced scenarios will require careful configuration to ensure authentication flows work as expected.

Challenges include:
- **Callback URLs** must match the configuration in the Azure App Registration and authentication settings.
- **Host Headers** might be overridden or mismatched due to App Gateway settings, leading to unexpected redirects.
- **Path-based routing rules** require careful configuration to avoid conflicts, particularly with wildcard entries or overlapping path patterns.

---

>[!IMPORTANT]
> From this point forward, this write-up highlights the particular challenge of handling a mix of authenticated and anonymous apps under a single domain using Azure App Gateway and App Service.

---

## App Service Authentication options with Path-Based Routing

![Path-Based Routing](images/pbrouting.png)

The diagram above illustrates a scenario where path-based routing was selected to facilitate rapid deployment of automated environments in an attempt to minimise domain and certificate management overhead. Here, multiple apps with varying authentication requirements—one requiring authentication and another allowing anonymous access—share an App Service plan to optimise costs.

Authentication can be handled using two options: easy auth and code-based authentication. In both options, the default App Service hostname is used to avoid domain conflicts. This is because you cannot configure the same custom domain for multiple App Services. See [Azure’s documentation](https://learn.microsoft.com/en-us/azure/app-service/manage-custom-dns-migrate-domain#how-do-i-migrate-a-domain-from-another-app:~:text=A%20domain%20name%20can%20be%20assigned%20to%20only%20one%20app%20in%20each%20deployment%20unit.) for more information.

>[!WARNING]
> Not using a custom domain on the App Service backend goes against general best practice. If you choose to experiment with using the default app service hostname instead of a customer domain, be sure to review the [Azure’s host name preservation guide](https://learn.microsoft.com/en-us/azure/architecture/best-practices/host-name-preservation) to verify the impact.

---
Lets look at the two options shown in the diagram.

### Option 1: Easy Auth with Host Header Override

Using Azure App Service’s Easy Auth with path-based routing requires configuring the host headers and authentication settings to ensure that requests respect the path and route configurations.

Note: The guidance below assume you have an existing Azure App Service with [easy auth](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization) enabled and an Azure Application Gateway set up with [path-based routing](https://learn.microsoft.com/en-us/azure/application-gateway/create-url-route-portal) rules.

#### Configuration:
1. **App Gateway Configuration**: Set up the backend HTTP setting in App Gateway to pick the hostname from the backend target. This uses the default App Service hostname to avoid domain conflicts. 

2. **App Registration Callback URL**: Ensure the app registration includes the correct callback URL, including the path (e.g., `https://yourdomain.com/yourpath/.auth/login/aad/callback`).

3. **Easy Auth Configuration (auth.json)**:
    - When using App Service Easy Auth behind Application Gateway, authentication redirects default to the app's Azure domain, often causing errors. To fix this, configure Easy Auth to read the X-Original-Host header from Application Gateway using file-based configuration as described in [Azure’s documentation](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-file-based#enabling-file-based-configuration). The documentation describes how you can create a file called auth.json to override the behaviour of App Service Authentication / Authorization for that site.

    - After creating the `auth.json` file, ensure you update it to define the required HTTP settings shown below::
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
    - Finally, make sure your app is configured to accept requests at the specified path. For Windows App Services, this can be done by mapping a [virtual directory](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=portal#map-a-url-path-to-a-directory) to the path. For Linux, this can be done by updating the app routes in your code.

>[!NOTE]
> One aspect that is not well-documented is the impact of the apiPrefix setting within file-based configuration. Without setting this value, Easy Auth will redirect to the root domain, disregarding your path-based routing configuration. To ensure correct routing, modify the apiPrefix value to match your app's path, as shown in the auth.json example above.

---

### Option 2: Code-Based Authentication with Host Header Override

For scenarios requiring more flexibility, implementing custom authentication within the app code can be beneficial. This allows for handling authentication requests programmatically, which can help with maintaining control over redirect paths and headers. With this approach the user can also be presented with a custom login page before accessing the app.

Note: The guidance below assume you have an existing Azure App Service and an Azure Application Gateway set up with [path-based routing](https://learn.microsoft.com/en-us/azure/application-gateway/create-url-route-portal) rules.

#### Configuration:
1. **Set up Authentication in Code**: Configure your app to authenticate users using a code-based approach with Microsoft Entra ID. Refer to the [Quickstart for Python Flask web app](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-web-app-python-flask?tabs=windows) to set up authentication with a sample app.

2. **App Registration Adjustments**: Ensure the redirect URI in the app registraion matches that of your app hostname and path. For the quickstart this will be 'https://YOURDOMAIN.COM/YOURPATH/getAToken'

3. **Path-Based Routing Adjustments**:
   - Update the app routes in app.py to ensure your app responds to the path set in App Gateway. Each route must be configured to accept requests at the specified path (e.g., `@app.route("/YOURPATH")`).
   - Define the `REDIRECT_PATH` in `app_config.py` with the format `/YOURPATH/getAToken`.

4. **Deployment**: Deploy the app to your Azure App Service using your CI/CD process or the [Azure Tools](https://code.visualstudio.com/docs/python/python-on-azure) extension in VSCode. 

---

### Options Summary and Considerations
App Service authentication with path-based routing options.

Choosing the right authentication approach depends on your specific requirements, trade-offs, and constraints. Below is a summary of the options discussed, along with key considerations to help ensure a smooth implementation.


| Approach                          | Pros                                 | Cons                                  |
|-----------------------------------|--------------------------------------|---------------------------------------|
| **Easy Auth**  | Minimal app code changes | Complex App Service configuration, limited flexibility |
| **Code-Based Auth**     | Full control, flexible, login page customisation               | Dependency on developer skills and code level changes |


With both options keep the following in mind:
1. **Host Headers and Redirect URLs**: Ensure that host headers match expected values to prevent redirect mismatches.
2. **Callback URL Consistency**: Confirm that callback URLs are consistently configured in both App Service and Entra ID to match the path-based routing setup.
3. **Avoid Overlapping Paths**: Be cautious with wildcard entries in routing (e.g., `/app*`), which may unintentionally capture routes for other applications.
4. **Health Probes**: When using App Gateway, configure health probes to target the correct path (e.g., `/YOUR_APP_PATH/`) and verify that App Service responds correctly.

---

## Conclusion

While path-based routing enables apps to share a common domain, it does introduce additional authentication complexities that shouldn't be considered lightly when comparing to the additional domain and certificate management of the subdomain approach. Managing callback URLs, host headers, and routing conflicts can complicate authentication workflows, especially when mixing authenticated and anonymous apps.

Given these challenges, using host-based routing (with subdomains) is often a simpler and more reliable solution for managing distinct apps. By assigning each app its own subdomain, you can maintain separate authentication configurations, reducing the risk of conflicts and simplifying management. Path-based routing can still be used effectively in scenarios where apps share common authentication requirements, but subdomains offer a more streamlined approach when dealing with varied app needs.

---

>[!IMPORTANT]
> To conclude and repeat my initial warning - This write-up focuses on more nuanced use cases, such as scenarios where you might be trying to achieve multiple authentication flows or a mix of anonymous and authentication-enabled apps behind a single domain name. Hopefully, it is clear that using subdomain routing is the simpler and more reliable option to consider first.
