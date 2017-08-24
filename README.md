# API Management OAuth App Services
Documentation taken from https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-protect-backend-with-aad and https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-protocols-oauth-client-creds and https://docs.microsoft.com/en-us/azure/app-service-api/app-service-api-authentication.

The scenario is about protecting a backend api with azure active directory authentication and requiring aad auth header in each request by the frontend application through the azure api management to the backend.

This has two different authentication schemes:
1. Blue internal authentication scheme for applications and users that are part of the same corp (and have identities in the same Azure Active Directory). For these requests we can create Bearer tokens that can be validated and passed trought API Management gateway and our Backend API. 
2. Green external authentication scheme for applications and users that are not part of our corp. For these requests we are validating their own JWT in the API Management Gateway and request and issue a new corp Bearer tokens and add them to the original request so that our backend can authenticate the call.

![](/img/arch.png)

To deploy this solution click on the button and make sure to do the preparation steps in advance.
Todo: Use KeyVault for secrets instead of environment variables.

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdenniszielke%2Fapim_lab%2Fmaster%2Farm%2Ftemplate.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>  

At the end you should end up with having the following resources provisioned:
![](/img/2017-08-22-15-50-58.png)

The code for the backend and frontend applications can be found here:
https://github.com/denniszielke/calc_backend
https://github.com/denniszielke/calc_frontend

## Preparation in Azure AD

You have to choose a deployment name for this scenario. This will ensure that you can deploy multiple times without running into naming collisions. This will also allow you to create the reply urls in advance. The recommendation would be to pick a two-letter- two-diggit combination like "dz11" for a deployname name.
In this case the frontend and backend applications will be generated under the following url:
``` 
https://{deployment_name}webbackend.azurewebsites.net (backend)
https://{deployment_name}webfrontend.azurewebsites.net (frontend)
https://{deployment_name}.portal.azure-api.net (developer portal)
https://{deployment_name}apiservice.api.net (api gateway)
```

Make sure you have Azure AD for your organisation set up. Either by creating a new one or using an existing one (requires for you to have permissions in it).

### Create Azure AD
Here's how to setup Azure Active Directory:
1. Navigate to https://manage.windowsazure.com and log in with the account that has an Azure subscription.
2. Click ACTIVE DIRECTORY management icon in the left pane.
3. Click NEW button at the bottom of the page.
4. Choose APP SERVICES > ACTIVE DIRECTORY > DIRECTORY > CUSTOM CREATE.
5. In the Add directory page, enter a name and domain name. 

### Create Service Principals
You need to create app registrations for the frontend application, the backend application and for the api management developer portal (so that you can test the backend api). Since you already picked the deployment name you can already set the homepage and reply urls:
![](/img/2017-08-22-17-44-45.png)

For the backend you can pick for the reply url:
```
https://{deployment_name}webbackend.azurewebsites.net/.auth/login/aad/callback
```

For the frontend you should create a key (and save it for later) and grant it delgated permissions to the backend application:

![](/img/2017-08-22-17-53-23.png)

![](/img/2017-08-22-17-52-33.png)

Remember to take note of all application ids, application uri id, tenant id of your azure active directory and the backend application secret.

![](/img/2017-08-22-18-30-11.png)

## Deployment

Click the following button, fill in the values and make sure that all special characters in the backend_app_uri_id are url encoded.

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdenniszielke%2Fapim_lab%2Fmaster%2Farm%2Ftemplate.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>  

The values would be in our case
``` 
backend_app_uri_id=https%3A%2F%2F{yourtenantname}.onmicrosoft.com%2Fdz11webfrontend
frontend_app_id=4d3d04ac-f811...
frontend_app_secret={yoursecret}
aad_tenant_id={yourtenantid}
```

## Post deployment

### Backend Authentication

Once the arm template has been successfully deployed. Go over to the portal. Open up your backend app service instance.
Go for "Authentication":
![](/img/2017-08-22-18-35-58.png)

Set the following values:
Client ID = Application Id of your backend app in Azure AD
IssuesUrl = https://sts.windows.net/{tenantid}/
Allowed Token Audiences = backend_app_uri_id

![](/img/2017-08-22-18-37-53.png)
Now try to access your backend app (with https - do not use http) you will get redirected and promted for aad credentials

### Create a subscription key for calculator product in API Management

Create a subscription.

Set validate jwt policy

<validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized. Access token is missing or invalid.">
    <openid-config url="https://login.microsoftonline.com/{yourtenant}.onmicrosoft.com/.well-known/openid-configuration" />
    <required-claims>
        <claim name="aud">
            <value>https://{yourtenant}.onmicrosoft.com/dz11webbackend</value>
        </claim>
    </required-claims>
</validate-jwt>

### Configure API Management Developer Portal to use 
https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-protect-backend-with-aad

### Call Backend API with subscription key from frontend

![](/img/2017-08-22-18-43-13.png)

### Check Application Insights

Both the frontend and the backend send a lot of telemetry and metrics to application insights.
Check it out.

![](/img/2017-08-22-18-49-59.png)

### Test calls using curl

Get a JWT for calling the backend from Azure AD
```
curl -X POST -H "Content-Type: application/x-www-form-urlencoded" -d 'client_id={caller application id}&resource={target app uri id}&client_secret={caller application secret}&grant_type=client_credentials' 'https://login.microsoftonline.com/{tenant id}/oauth2/token'
```

Post that token either to the backend or the APIM gateway

```
curl -X GET -H "Ocp-Apim-Subscription-Key: {herebesubscriptionkey}" -H "Authorization: Bearer {here be access_token}" 'https://{here be api url}'
```

## External authentication

For validation of external tokens see the official apim documentation:
https://docs.microsoft.com/de-de/azure/api-management/api-management-access-restriction-policies#ValidateJWT 

Now assume we have validated the external token and we want to attach our own corp issued token to the original request.
We first have to set a couple of variables to be stored as named properties inside the api management. You can find the documentation for this here: https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties 

![](/img/2017-08-24-08-44-58.png)

We also improve the performance of this by storing authentication token inside the cache. You can optimize this by having different cache keys for different subscriptions keys.
Now that these values are available we can reference them inside our external product policy:

``` 
<policies>
    <inbound>
        <cache-lookup-value key="calc_be" default-value="" variable-name="calc_be_token"/>
        <choose>
            <when condition="@(string.IsNullOrEmpty((string) context.Variables["calc_be_token"]))">
                <!--          Extract Token from Authorization header parameter and load variables         -->
                <set-variable name="token" value="@(context.Request.Headers.GetValueOrDefault("Authorization","scheme param").Split(' ').Last())"/>
                <set-variable name="tenant-id" value="{{tenant-id}}"/>
                <set-variable name="impersonation-app-id" value="{{impersonation-app-id}}"/>
                <set-variable name="impersonation-app-secret" value="{{impersonation-app-secret}}"/>
                <set-variable name="resource" value="{{resource}}"/>
                <!--          Send request to receive token          -->
                <send-request mode="new" response-variable-name="tokenstate" timeout="20" ignore-error="true">
                    <set-url>@("https://login.microsoftonline.com/" + (string)context.Variables["tenant-id"] + "/oauth2/token")</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/x-www-form-urlencoded</value>
                    </set-header>
                    <set-body>@("client_id=" + (string)context.Variables["caller-id"] + "&resource=" + (string)context.Variables["dcscalcbackend-idurl"] + "&client_secret=" + (string)context.Variables["caller-secret"]+ "&grant_type=client_credentials")
    </set-body>
                </send-request>
                <!--          Store the access token to a variable          -->
                <set-variable name="access_token" value="@(((IResponse)context.Variables["tokenstate"]).Body.As<JObject>()["access_token"].ToString())"/>
                    <!--        Get Token and set in request and cache      -->
                    <set-header name="Authorization" exists-action="override">
                        <value>@("Bearer " + (string)context.Variables["access_token"])</value>
                    </set-header>
                    <cache-store-value key="calc_be" value="@((string)context.Variables["access_token"])" duration="30"/>
                </when>
                <otherwise>
                    <!--       If there is already an access token in the cache we use that one      -->
                    <set-header name="Authorization" exists-action="override">
                        <value>@("Bearer " + (string)context.Variables["calc_be_token"] )</value>
                    </set-header>
                </otherwise>
            </choose>
            <base/>
        </inbound>
        <backend>
            <base/>
        </backend>
        <outbound>
            <base/>
        </outbound>
    </policies>

```