# API Management OAuth App Services
Documentation taken from https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-protect-backend-with-aad and https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-protocols-oauth-client-creds and https://docs.microsoft.com/en-us/azure/app-service-api/app-service-api-authentication.

The scenario is about protecting a backend api with azure active directory authentication and requiring aad auth header in each request by the frontend application through the azure api management to the backend.

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