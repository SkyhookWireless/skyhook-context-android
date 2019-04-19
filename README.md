# Skyhook Context Accelerator (Android)

## Prerequisites

Supported on all Android versions starting with 2.3.x (Gingerbread), including forked platforms such as the Kindle Fire.

## Installation

### Add SDK to your project

Add Skyhook's Maven repository URL to the `repositories` section in your `build.gradle`:
```gradle
repositories {
    maven { url 'https://skyhookwireless.github.io/skyhook-context-android' }
}
```
Add SDK to the `dependencies` section:
```gradle
dependencies {
    implementation 'com.skyhook.context:accelerator:2.1.4'
}
```
Note that you can exclude transitive dependencies to resolve version conflicts, and include those dependencies separately:
```gradle
implementation 'com.android.support:appcompat-v7:28.0.0'
implementation('com.skyhook.context:accelerator:2.1.4') {
    exclude module: 'support-v4'
}
```

### API key

Put your Skyhook API key within the `<application>` element in `AndroidManifest.xml`:
```xml
<meta-data
    android:name="com.skyhook.context.API_KEY"
    android:value="PUT YOUR KEY HERE"/>
```
You can obtain the API key from [my.skyhookwireless.com](https://my.skyhookwireless.com).

### Permissions

Accelerator SDK automatically adds the following permissions to your app's manifest:

| Android Permission                                     | Used For
|--------------------------------------------------------|---------
| android.permission.INTERNET                            | Communication with Skyhook's servers
| android.permission.CHANGE_WIFI_STATE                   | Initiation of Wi-Fi scans
| android.permission.ACCESS_WIFI_STATE                   | Obtaining information about the Wi-Fi environment
| android.permission.ACCESS_COARSE_LOCATION              | Obtaining Wi-Fi or cellular based locations
| android.permission.ACCESS_FINE_LOCATION                | Accessing GPS location for hybrid location functionality
| android.permission.WAKE_LOCK                           | Keeping processor awake when receiving background updates
| android.permission.ACCESS_NETWORK_STATE                | Checking network connection type to optimize performance
| android.permission.RECEIVE_BOOT_COMPLETED              | Resuming monitoring after device reboot
| android.permission.FOREGROUND_SERVICE                  | Obtaining location in background mode
| com.google.android.gms.permission.ACTIVITY_RECOGNITION | Determining user activity type to optimize performance

The `RECEIVE_BOOT_COMPLETED` permission is optional. If you don't want to automatically resume monitoring after reboot, you can exclude the permission in your `AndroidManifest.xml`:
```xml
<manifest ... xmlns:tools="http://schemas.android.com/tools">
    ...
    <uses-permission
        android:name="android.permission.RECEIVE_BOOT_COMPLETED"
        tools:node="remove"/>
    ...
</manifest>
```

The `FOREGROUND_SERVICE` permission may also be excluded depending on your campaign configuration and accuracy requirements. Please contact Skyhook support for more details.

### Google Play Services

Accelerator SDK uses the following Google Play Services APIs by default:

| Google Play Services API                      | Used For
|-----------------------------------------------|---------
| com.google.android.gms:play-services-location | Accessing location, geofencing and activity recognition for optimizing performance|
| com.google.android.gms:play-services-ads      | Obtaining Google Ad Id for user personification|

If your application is using Google Play Services on its own it is recommended to align versions of all dependent `play-services-xxx` modules to avoid conflicts at compile time or runtime.

For example, if your app is using `play-services-maps:16.0.0`, add the following dependencies in your `build.gradle`:
```gradle
dependencies {
    implementation 'com.google.android.gms:play-services-maps:16.0.0'
    implementation 'com.google.android.gms:play-services-location:16.0.0'
    runtimeOnly 'com.google.android.gms:play-services-ads:16.0.0'
    ...
}
```

Note that the oldest version of Google Play Services supported by Accelerator SDK is 8.1.0.

#### Opting out of Google Ad Id collection

If you want to opt out of Google Ad Id collection or user personification in general, you can do so by calling the `setOptedIn` or `setUserId` methods. See more details in the [Privacy Considerations](#privacy-considerations) section.

If you also want to remove the dependency to the `play-services-ads` module in your app entirely, you can do so in your `build.gradle`:
```gradle
implementation('com.skyhook.context:accelerator:2.1+') {
    exclude module: 'play-services-ads'
}
```

### Using the Android Emulator

The Context Accelerator SDK will not be able to determine location using Wi-Fi or cellular beacons from the emulator because it is unable to scan for those signals. Because of that, its functionality will be limited on the emulator. In order to verify your integration of the SDK using the emulator, you may want to use the `requestIPLocation()` method call. The full functionality will work only on an Android device.

## Initializing

### Import the SDK

Import the accelerator API:
```java
import com.skyhook.context.Accelerator;
```

### Initialize Accelerator API

Add the following call in the `onCreate` method of your activity or application class:
```java
Accelerator.init(this);
```

### Request location permission

With the new permissions model introduced in Android M, the Context Accelerator requires location permission to be granted before calling most of its methods. Depending on how the Context Accelerator is used in the application, developer can decide when to request the permission and if an explanation needs to be displayed for the user:
```java
ActivityCompat.requestPermissions(
    this, new String[] { Manifest.permission.ACCESS_FINE_LOCATION }, 0);
```

## Skyhook Personas

You can request a refresh of the Skyhook persona of the current user by calling `refreshPersona()`:
```java
Accelerator.refreshPersona()
           .setOnSuccessListener(new OnSuccessListener<Persona>() {
               @Override
               public void onSuccess(final Persona persona) {
                   // process persona...
               }
           })
           .setOnFailureListener(new OnFailureListener() {
               @Override
               public void onFailure(final int statusCode) {
                   // handle failure...
               }
           });
```

### Latest Persona

You can access the latest cached persona at any time by calling the `getLastPersona()` in a similar way to `refreshPersona()`:
```java
Accelerator.getLastPersona()
           .setOnSuccessListener(new OnSuccessListener<Persona>() {
               @Override
               public void onSuccess(final Persona persona) {
                   // process persona...
               }
           })
           .setOnFailureListener(new OnFailureListener() {
               @Override
               public void onFailure(final int statusCode) {
                   // handle failure...
               }
           });
```

Note that for apps using Google Play Services the latest persona may be erased after a change in the Google Ad settings on the device. The `onFailure` method will be called with the `ERROR_PERSONA_NOT_SET` status code. You may want to request a persona refresh in that case.

The `Persona` object has the following demographic properties:

- age
- gender
- ethnicity
- income
- education

Each of these properties is a `List` composed of `Demographic` objects, listed in a decreasing order of probability.

For example, to list all the values for the current ethnicity persona:
```java
for (Demographic demographic : persona.ethnicity) {
    Log.d(TAG, "value:" + demographic.value
               + " probability:" + demographic.probability
               + " variance:" + demographic.variance);
}
```

The result of the above code would print this:
```
value:white probability:0.65 variance: 0.1
value:other probability:0.15 variance: 0.1
value:black probability:0.12 variance: 0.1
value:asian probability:0.08 variance: 0.1
```

Note that since the values are ordered by probability, you can simply grab the first element (index 0) from the list to get the most probable value for the current user.

### Behaviors

In addition to the five demographic objects, `Persona` also includes a `behaviors` field.
```java
for (Behavior behavior : persona.behaviors) {
    Log.d(TAG, "id:" + behavior.id + " name:" + behavior.name);
}
```

The result of the above code would print something like this:
```
ID:12341318394918 name:auto intenders
ID:1234131839491234 name:auto enthusiasts
```

## Campaign Monitoring

### Enable Monitoring

Once the location permission has been granted, you can start monitoring campaigns:
```java
Accelerator.startMonitoringForAllCampaigns();
```

For most apps the `startMonitoringForAllCampaigns()` method is all that is needed for starting monitoring, but if more control is required, then your app can instead start monitoring for individual campaigns:
```java
Accelerator.startMonitoringForCampaign("YourCampaignName1");
Accelerator.startMonitoringForCampaign("YourCampaignName2");
Accelerator.startMonitoringForCampaign("YourCampaignName3");
```

When monitoring for individual campaigns, you can stop monitoring ones that are no longer needed:
```
Accelerator.stopMonitoringForCampaign("YourCampaignName2");
```

**Note that if all campaigns are being monitored, then calling `stopMonitoringForCampaign()` has no effect. It will not stop the specified campaign from being monitored.**

Or you can stop monitoring for all campaigns, whether individual campaigns or all of them are being monitored:
```java
Accelerator.stopMonitoringForAllCampaigns();
```

### Register for events

In order to handle campaign events during monitoring, you need to add a broadcast receiver to your application. The receiver will be notified when the user enters or exits campaign venues even when the app is in the background.

Add the following line in your `AndroidManifest.xml`:
```xml
<uses-permission android:name="com.skyhook.context.RECEIVE_EVENT"/>
```

Declare a receiver with the intent filter set to the `ACCELERATOR_EVENT` action:
```xml
<receiver android:name=".MyAcceleratorReceiver"
          android:exported="false">
    <intent-filter>
        <action android:name="com.skyhook.context.ACCELERATOR_EVENT"/>
    </intent-filter>
</receiver>
```

Implement the broadcast receiver for handling campain venue transitions:
```java
public void onReceive(Context context, Intent intent) {
    if (Accelerator.hasError(intent)) {
        int errorCode = Accelerator.getErrorCode(intent);
        // handle error
    } else {
        CampaignVenue venue = Accelerator.getTriggeringCampaignVenue(intent);
        if (venue != null) {
            int transition = Accelerator.getCampaignVenueTransition(intent);
            // handle transition
        }
    }
}
```

## IP Location

The Context Accelerator SDK can also be used to give you on-demand IP locations using the requesting remote IP:
```java
Accelerator.requestIPLocation()
           .setOnSuccessListener(new OnSuccessListener<Location>() {
               @Override
               public void onSuccess(final Location location) {
                   // Note that location will never be null
                   double latitude = location.getLatitude();
                   double longitude = location.getLongitude();
                   long systemTime = location.getTime();

                   // Note that extras will never be null
                   Bundle extras = location.getExtras();

                   // This extra is always present
                   long elapsedRealtimeMillis = extras.getLong("elapsedRealtimeMillis");

                   // process latitude, longitude, systemTime, and elapsedRealtimeMillis...

                   if (location.hasAccuracy()) {
                       float accuracy = location.getAccuracy();
                       // process accuracy...
                   }

                   if (location.hasAltitude()) {
                       double altitude = location.getAltitude();
                       // process altitude...
                   }

                   IPLocationType type = (IPLocationType) extras.get("type");
                   if (type != null && type != IPLocationType.UNKNOWN) {
                       // process IP type...
                   }

                   StreetAddress streetAddress = (StreetAddress) extras.get("streetAddress");
                   if (streetAddress != null) {
                       String postalCode = streetAddress.postalCode;
                       if (postalCode != null) {
                           // process postal code...
                       }
                   }
               }
           })
           .setOnFailureListener(new OnFailureListener() {
               @Override
               public void onFailure(final int code) {
                   // handle IP location failure...
               }
           });
```

If successful, the `onSuccess()` method will be called with an Android `Location` object which contains some extras that are specific to IP locations. All IP locations generated by accelerator are guaranteed to have a valid latitude, longitude, and timestamp (both UTC time and elapsed real-time in milliseconds, not nanoseconds, since boot). All other properties are optional.

The IP location properties are as follows:

Method Name      | Property Type | Definition
-----------------|---------------| ----------
getLatitude()    | double        | latitude, in degrees
getLongitude()   | double        | longitude, in degrees
getTime()        | long          | UTC time of this fix, in milliseconds since January 1, 1970

Required extras:

Extra Name            | Property Type | Definition
----------------------|---------------| ----------
elapsedRealtimeMillis | long          | time of this fix in milliseconds, in elapsed real-time since system boot

Optional properties:

Method Name   | Property Type | Definition
--------------|---------------| ----------
getAccuracy() | float         | horizontal accuracy if available, in meters
getAltitude() | double        | altitude if available, in meters above sea level

Optional extras:

Extra Name    | Property Type                     | Definition
--------------|-----------------------------------| ----------
type          | [IPLocationType](#iplocationtype) | type of IP address if available
streetAddress | StreetAddress                     | street address if available, currently only postalCode is populated

IP location types:

Value   | Definiton
--------|----------
FIXED   | fixed IP address
MOBILE  | mobile IP address
UNKNOWN | unable to resolve type of IP

## Venue Information

The Context Accelerator SDK provides a collection of methods for obtaining venue information related to your campaigns and location.

### Nearby Monitored Venues

The `getNearbyMonitoredVenues` method allows the client to obtain the unique identifiers of nearby venues that are part of actively monitored campaigns. This method can be used in conjunction with the `getVenueInfo` method to obtain more detailed venue information:
```java
Accelerator.getNearbyMonitoredVenues(100)
           .setOnSuccessListener(new OnSuccessListener<List<NearbyCampaignVenue>>() {
               @Override
               public void onSuccess(final List<NearbyCampaignVenue> venues) {
                   // handle nearby venues...
               }
           })
           .setOnFailureListener(new OnFailureListener() {
               @Override
               public void onFailure(final int code) {
                   // handle error...
               }
           });
```

### Venue Information by unique identifer

The `getVenueInfo` method allows the client to obtain more detailed venue information using the unqiue venue identifiers from the `CampaignVenue` and `NearbyCampaignVenue` objects:
```java
@Override
public void onSuccess(final List<NearbyCampaignVenue> venues) {
    // Fetch venue information for nearby venues

    List<Long> ids = new ArrayList<>(venues.size());
    for (NearbyCampaignVenue venue : venues)
        ids.add(venue.venueId);

    Accelerator.getVenueInfo(ids)
               .setOnSuccessListener(new OnSuccessListener<List<VenueInfo>>() {
                   @Override
                   public void onSuccess(final List<VenueInfo> venueInfo) {
                       // handle venue information...
                   }
               })
               .setOnFailureListener(new OnFailureListener() {
                   @Override
                   public void onFailure(final int code) {
                       // handle error...
                   }
               });

    // ...
}
```

### Venue Information at the current location

The `getVenueInfoAtLocation()` method allows the client to request the venue information at the current user location:
```java
Accelerator.getVenueInfoAtLocation()
           .setOnSuccessListener(new OnSuccessListener<List<VenueInfo>>() {
               @Override
               public void onSuccess(final List<VenueInfo> venueInfo) {
                   // handle venue information...
               }
           })
           .setOnFailureListener(new OnFailureListener() {
               @Override
               public void onFailure(final int code) {
                   // handle error...
               }
           });
```

## Privacy Considerations

For apps using Google Play Services, the Context Accelerator SDK will collect certain usage information in order to improve the quality of Skyhook's positioning and context products. For a detailed overview of the data that the Skyhook Context Accelerator SDK will collect and use, please read our [Privacy Policy](http://www.skyhookwireless.com/privacy-policy/skyhook). Please note that for apps not using Google Play Services, user-based personas are not currently supported and the Accelerator SDK will only collect and provide anonymous location information.

To leverage user-based persona information, the Accelerator SDK needs an anonymous identifier. By default it will use Google Ad Id. You can override the identifier by assigning a custom identifier using Accelerator's `setUserId()` method:
```java
Accelerator.setUserId("some-valid-and-unique-user-id");
```

In addition to Skyhook's default privacy protections, developers integrating the Context Accelerator SDK may also allow application users to opt out of usage data collection. Note that the application may receive degraded context information if data collection is disabled.

To disable usage data collection:
```java
Accelerator.setOptedIn(false);
```

You can check the data collection status anytime with:
```java
Accelerator.isOptedIn()
           .setOnSuccessListener(new OnSuccessListener<Boolean>() {
               @Override
               public void onSuccess(final Boolean optedIn) {
                   // ...
               }
           });
```

or re-enable it with:
```java
Accelerator.setOptedIn(true);
```

Note that by default Accelerator SDK follows Google Ads settings: data collection will be enabled only if Google Play Services are installed on the device, the application uses the Google Play Services library and the `Opt out of interest-based ads` option is unchecked in Google Settings.

If your application is not using the Google Play Services library but the application and/or content provider has received an EXPLICIT opt in from the user to leverage a unique device identifier for Geofence logging and personification, developers must set the user ID and set the opt-in to `true` to enable data collection:
```java
Accelerator.setUserId("some-valid-and-unique-user-id");
Accelerator.setOptedIn(true);
```

If either of these conditions are not met, the default behavior of the SDK is to treat the user as opted out and disable data collection on the server.

For more details on how to configure usage of Google Play Services in your app see the [Google Play Services](#google-play-services) section.
