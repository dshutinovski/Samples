﻿# CloudFoundry Single Signon Security Sample App 
ASP.NET Core sample app illustrating how to make use of the Steeltoe [CloudFoundry External Security Provider](https://github.com/SteeltoeOSS/Security) for Authentication and Authorization against a CloudFoundry OAuth2 security service (e.g. [UAA Server](https://github.com/cloudfoundry/uaa) or [Pivotal Single Signon](https://docs.pivotal.io/p-identity/)).

# Pre-requisites - CloudFoundry

1. Install Pivotal CloudFoundry 1.7
2. Install .NET Core SDK
3. Optionally, Single Signon for CloudFoundry if you wish use it as your OAuth Security server.
4. Web tools installed and on Path. If you have VS2015 Update 3 installed then add this to your path: C:\Program Files (x86)\Microsoft Visual Studio 14.0\Web\External

# Create OAuth2 Service Instance on CloudFoundry
You must first create an instance of a OAuth2 service in a org/space. As mentioned above there are a couple to choose from. In the steps that follow we will directly use the [UAA Server](https://github.com/cloudfoundry/uaa) as an OAuth2 service.

If instead, you want to use the [Pivotal SSO](https://docs.pivotal.io/p-identity/) service, follow the installation and configuration instructions [here](https://docs.pivotal.io/p-identity/installation.html). Note that you will still need to add a User and Group `testgroup` as described below to the [SSO Internal Store](http://docs.pivotal.io/p-identity/configure-id-providers.html#add-to-int).
### Install UAA Command Line
Before creating the OAuth2 service instance, we first need to use the UAA command line tool to establish some security credentials for our sample app. To install the UAA command line tool and target it to your UAA server:

1. Install Ruby if you don't already have it.
2. gem install cf-uaac
3. uaac target uaa.`YOUR-CLOUDFOUNDRY-SYSTEM-DOMAIN` (e.g. `uaac target uaa.system.testcloud.com`)

### Obtain Admin Client Access Token 
Next we need to authenticate and obtain an access token for the `admin client` from the UAA server so that we can add our new application/user credentials. You will need the `Admin Client Secret` for your installation of CloudFoundry in order to accomplish this. If you are using Pivotal CloudFoundry (PCF), you can obtain this from the `Ops Manager/Elastic Runtime` credentials page under the `UAA` section.  Look for `Admin Client Credentials` and then use it as follows:

1. uaac token client get admin -s `ADMIN_CLIENT_SECRET`
2. uaac contexts

### Add User and Group
Next we will add a new `user` and `group` to the UAA Server database. Do NOT change the groupname: `testgroup` as that is used for policy based authorization in the sample application. Of course you can change the username and password to anything you would like.

1. uaac group add testgroup
2. uaac user add dave --given_name Dave --family_name Tillman --emails dave@testcloud.com --password Password1!
3. uaac member add testgroup dave 

### Add New Client for our App
Once complete we are ready to add our application as a new client to the UAA server. This will establish our applications credentials and enable it to interact with the UAA server. To do this you can use the line below, but you must replace the `YOUR-CLOUDFOUNDRY-APP-DOMAIN` with your setups domain.

1. uaac client add myTestApp --scope cloud_controller.read,cloud_controller_service_permissions.read,openid,testgroup --authorized_grant_types authorization_code,refresh_token --authorities uaa.resource --redirect_uri http://single-signon.`YOUR-CLOUDFOUNDRY-APP-DOMAIN`/signin-cloudfoundry --autoapprove cloud_controller.read,cloud_controller_service_permissions.read,openid,testgroup --secret myTestApp
 
### Add CUPs based OAuth Service
Last, we create a CUPS service providing the appropriate UAA server configuration data. You can use the provided `credentials.json` file when creating your CUPS service, but you will FIRST need to edit it and replace the `YOUR-CLOUDFOUNDRY-SYSTEM-DOMAIN` with your setups domain. Once done you can do the following:

1. cf target -o myorg -s development
2. cf cups myOAuthService -p credentials.json


# Publish App & Push to CloudFoundry

1. cf target -o myorg -s development
2. cd samples/Security/src/CloudFoundrySingleSignon
3. dotnet restore --configfile nuget.config
4. Publish app to a directory  
(e.g. `dotnet publish --output $PWD/publish --configuration Release --framework net451 --runtime win7-x64`)
5. Push the app using the provided manifest.
 (e.g.  `cf push -f manifest-windows.yml -p $PWD/publish` or `cf push -f manifest.yml -p $PWD/publish` )

Note: The provided manifest(s) will create an app named `single-signon` and attempt to bind it to the CUPS service `myOAuthService`.

Note: We have experienced this [problem](https://github.com/dotnet/cli/issues/3283) when using the RTM SDK and when publishing to a relative directory... workaround is to use full path.

# What to expect - CloudFoundry
After building and running the app, you should see something like the following in the logs. 

To see the logs as you startup and use the app: `cf logs single-signon`

On a Windows cell, you should see something like this during startup:
```
2016-08-05T07:23:02.15-0600 [CELL/0]     OUT Creating container
2016-08-05T07:23:03.81-0600 [CELL/0]     OUT Successfully created container
2016-08-05T07:23:09.07-0600 [APP/0]      OUT Running cmd /c .\CloudFoundrySingleSignon --server.urls http://*:%PORT%
2016-08-05T07:23:14.68-0600 [APP/0]      OUT Hosting environment: development
2016-08-05T07:23:14.68-0600 [APP/0]      OUT Content root path: C:\containerizer\75E10B9301D2D9B4A8\user\app
2016-08-05T07:23:14.68-0600 [APP/0]      OUT Application started. Press Ctrl+C to shut down.
2016-08-05T07:23:14.68-0600 [APP/0]      OUT Now listening on: http://*:51217
```
At this point the app is up and running.  You can access it at http://single-signon.`YOUR-CLOUDFOUNDRY-APP-DOMAIN`/.  

On the apps menu, click on the `Log in` menu item and you should be redirected to the CloudFoundry login page. Enter `dave` and `Password1!`, or whatever name/password you used above,  and you should be authenticated and redirected back to the single-signon home page.

Two of the endpoints in the `HomeController` have Authorization policys applied:
```
[Authorize(Policy = "testgroup")]
public IActionResult About()
{
    ViewData["Message"] = "Your application description page.";

    return View();
}


[Authorize(Policy = "testgroup1")]
public IActionResult Contact()
{
    ViewData["Message"] = "Your contact page.";

    return View();
}
```
If you try and access the `About` menu item you should see the `About` page as user `dave` is a member of that group and is authorized to access the end point.

If you try and access the `Contact` menu item you should see `Access Denied, Insufficent permissions` as `dave` is not a member of the `testgroup1` and therefore can not access the end point.

If you access the `InvokeJwtSample` menu item, you will find the app will attempt to invoke a secured endpoint in a second Security sample app [CloudFoundryJwtAuthentication](https://github.com/SteeltoeOSS/Samples/tree/dev/Security/src/CloudFoundryJwtAuthentication). In order for this to be functional you must first push the [CloudFoundryJwtAuthentication](https://github.com/SteeltoeOSS/Samples/tree/dev/Security/src/CloudFoundryJwtAuthentication) sample using the Readme instructions you can find [here](https://github.com/SteeltoeOSS/Samples/tree/dev/Security/src/CloudFoundryJwtAuthentication).

Once you have [CloudFoundryJwtAuthentication](https://github.com/SteeltoeOSS/Samples/tree/dev/Security/src/CloudFoundryJwtAuthentication) up and running, then if you access the `InvokeJwtSample` menu item when you are logged in, you should see some `values` returned from the [CloudFoundryJwtAuthentication](https://github.com/SteeltoeOSS/Samples/tree/dev/Security/src/CloudFoundryJwtAuthentication) app.  If you are logged out, then you will see a `401 (Unauthorized)` message.
