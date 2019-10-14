# Azure Pipelines Demo

This repo is a sample project to help developers learn basic DevOps practices. It largely consists of a generic sample Xamarin.Forms project. The included YAML definition for Azure Pipelines can be used as a template for configuring your build pipelines for your Xamarin projects whether they are based on Xamarin Native, Xamarin.Forms, or Uno.

## Getting Started

Before getting started with this project be sure to read the Blog Post on [Getting Started with Azure Pipelines for Xamarin Developers](https://dansiegel.net/post/2019/08/05/getting-started-with-azure-pipelines-for-xamarin-developers).

## About the DevOps

When setting up your DevOps Pipeline there is a little bit of prep work you will need to do to ensure you have the proper resources available during the build.

### Library Resources for iOS

Apple requires that we have a code signing certificate they have issued us for either Development or Distribution and we need ensure that these resources are available for the build. To accomplish this be sure to upload your Develoment and Distribution certificate and Provisioning Profile's as Secure Files in the Azure DevOps Project. To keep things tidy you will want to create a Variable Group called `iOS-Signing`. Be sure to add variables for both QA and the Store and set some appropriate values.

| QA | Store | Example Value |
|:--:|:--:|:--:|
| iOSDevelopmentCertificate | iOSDistributionCertificate | ios-dev.p12 |
| iOSDevelopmentPassword | iOSDistributionPassword | {your password} |
| iOSDevelopmentProvisioningProfile | iOSDistributionProvisioningProfile | Wildcard_Development.mobileprovision |

Note that the values for the Certificates and Provisioning Profile's will just be the file names. You will want to upload them to the Secure Files.

### Library Resources for Android

Note that while the build Pipeline here has a dependency on a Variable Group called `Android-Signing`, it really is not needed as Google requires that they have your Signing Key to sign the generated APK's from your Android App Bundle (.aab). As a result we aren't currently using these variables in the build and they are left here purely for those who wish to continue shipping APK's directly.

- KeystoreFileName
- KeystoreName
- KeystorePassword

### Common Library Resources

This build uses the [Mobile.BuildTools](https://github.com/dansiegel/Mobile.BuildTools) to inject environment specific secrets or configuration settings. As a result we use some common settings here this is provided with a Variable Group named `DemoDevOps-Secrets` which has the following variables.

- AppCenterKey_Android_QA
- AppCenterKey_Android_Store
- AppCenterKey_iOS_QA
- AppCenterKey_iOS_Store
- SampleString

## Distribution

Distribution here is done very differently. It is assumed that we are using App Center for Diagnostics and Analytics. As part of this we need to ensure that App Center has the DSYM for each build, while we do not have such a concern on Android.

### Distribution for Android

To distribute our Android application to the store with our Android App Bundle (.aab) we need to use a new Task that comes from the Google Play extension for Azure DevOps which you must first make sure you've installed from the [Marketplace](https://marketplace.visualstudio.com).

```yaml
- task: GooglePlayReleaseBundle@3
  inputs:
    serviceConnection: 'GooglePlay'
    applicationId: 'com.avantipoint.demodevops'
    bundleFile: 'Droid-Store/com.avantipoint.demodevops-Signed.aab'
    track: 'internal'
    rolloutToUserFraction: true
```

Some things to note... you will need to set up a Service Account in the Google Play console and set this up as a Service Connection for the Azure DevOps Project. Here we have called the Service Connections `GooglePlay`. The `applicationId` naturally should be whatever your app's id is. Finally note that we have specified the path to our `.aab` looking inside the Store built artifact and we are pushing this to the `internal` track with full User Rollout.

### Distribution for iOS

We certainly could push directly to the Apple App Store from Azure DevOps with the installable extension in the Marketplace. Since we want to use App Center for Analytics and Diagnostics we need to be sure that App Center has the DSYM. As currently configured we will simply be pushing from Azure DevOps to App Center and from there we can manually be pushing builds to the Store. For actual production apps though we can help automate this process and upload builds to the App Store where we can then decide what to do with the builds. To change this you will want to add the following two properties to the inputs of task to upload the Store build to App Center.

```yaml
  inputs:
    destinationType: store
    destinationStoreId: {the Store Guid from the App Center API}
```

Note here that we show you need a Guid for the Store Channel you want to push to. In order to do this you'll need to use Postman or another client to call App Center's API. In order to get what you need you'll do a **GET** request as follows:

```http
GET /v0.1/apps/{accountName}/{appName}/distribution_stores HTTP/1.1
Host: api.appcenter.ms
X-API-Token: {your App Center API Token}
```

After making the call you'll get back an arrary that will look something like the following. You'll want to choose which track you would like to distribute to and then copy the `id`. You'll use that id for the `destinationStoreId` in the YAML.

```json
[
    {
        "id": "{What you need}",
        "name": "App Store Connect Users",
        "type": "apple",
        "track": "testflight-internal",
        "created_by": "{doesn't matter}",
        "service_connection_id": "{doesn't matter}"
    },
    {
        "id": "{What you need}",
        "name": "Production",
        "type": "apple",
        "track": "production",
        "created_by": "{doesn't matter}",
        "service_connection_id": "{doesn't matter}"
    }
]
```