# Estimote Proximity SDK for Android 

*Estimote Proximity SDK aims to provide a simple way for apps to react to physical context by reading signals from Estimote Beacons.*

**Main features:**

1. Reliability. It's built upon Estimote Monitoring, Estimote's algorithm for reliable enter/exit reporting.
2. No need to operate on abstract identifiers, or Proximity UUID, Major, Minor triplets. Estimote Proximity SDK lets you define zones by setting predicates for human-readable JSONs.
3. You can define multiple zones for a single beacon, i.e. at different ranges.
4. Cloud-backed grouping. When you change your mind and want to replace one beacon with another, all you need to do is reassign JSON attachments in Estimote Cloud. You don't even need to connect to the beacon!

## Requirements

- One or more [Estimote Proximity or Location Beacon](https://estimote.com/products/) with the `Estimote Location` packet advertising enabled. 
- An Android device with Bluetooth Low Energy support. We suggest using Android 5.0+ (Lollipop or newer). 
- An account in [Estimote Cloud](https://cloud.estimote.com/#/)
## Installation

Add this line to your `build.gradle` file:
```Gradle
implementation 'com.estimote:proximity-sdk:0.2.0'
```
> If you are using Gradle version below `3.0.0` then you should use `compile` instead of `implementation`.

## Attachment-based identification explanation

Details of each of your Estimote devices are available in Estimote Cloud. Each device has a unique identifier, but remembering it and using it for every one of your devices can be challenging. This is why Estimote Proximity SDK uses attachment-based identification.

Each device has an associated attachment. When the SDK detects a change in the proximity to a device, it checks the device's attachment to see which of the registered rules should be applied.

## 0. Setting up attachments in your Estimote Cloud account
1. Go to https://cloud.estimote.com/#/
2. Click on the beacon you want to configure
3. Click the Edit settings button
4. Click the Beacon Attachment tab
5. Add any attachment key-value pair you want
7. Click Save changes
Attachments are Cloud-only settings — no additional connecting to the beacons with the Estimote app is required!

![Cloud attachments](/images/adding_attachment_json_tag.png)

## 1. Build proximity observer
The `ProximityObserver` is the main object for performing proximity observations. Build it using `ProximityObserverBuilder` - and don't forget to put in your Estimote Cloud credentials!

```Kotlin
// Kotlin
val cloudCredentials = EstimoteCloudCredentials(YOUR_APP_ID_HERE , YOUR_APP_TOKEN_HERE)
val proximityObserver = ProximityObserverBuilder(applicationContext, cloudCredentials)
                .withBalancedPowerMode()
                .withOnErrorAction { /* handle errors here */ }
                .build()
```

```Java
// Java
EstimoteCloudCredentials cloudCredentials = new EstimoteCloudCredentials(YOUR_APP_ID_HERE, YOUR_APP_TOKEN_HERE);
ProximityObserver proximityObserver = new ProximityObserverBuilder(getApplicationContext(), cloudCredentials)
                .withBalancedPowerMode()
                .withOnErrorAction(new Function1<Throwable, Unit>() {
                  @Override
                  public Unit invoke(Throwable throwable) {
                    return null;
                  }
                })
                .build();
```
You can customize your `ProximityObject` using the available options:
- **withLowLatencyPowerMode** - the most reliable mode, but may drain battery a lot. 
- **withBalancedPowerMode** - balance between scan reliability and battery drainage. 
- **withLowPowerMode** - battery efficient mode, but not that reliable.
- **withOnErrorAction** - action triggered when any error occurs - such as cloud connection problems, scanning, etc.
- **withScannerInForegroundService** - starts the observation proces with scanner wrapped in [foreground service](https://developer.android.com/guide/components/services.html). This will display notification in user's notifications bar, but will ensure that the scanning won't be killed by the system. **Important:** Your scanning will be handled without the foreground service by default. 
- **withTelemetryReporting** - Enabling this will send telemetry data from your beacons, such as light level, or temperature, to our cloud. This is also an important data for beacon health check (such as tracking battery life for example). Bear in mind that enabling this will slightly increase battery drain. 
- **withAnalyticsReportingDisabled** - Analytic data (current visitors in your zones, number of enters, etc) ) is sent to our cloud by default. Use this to turn it off.

## 2. Define proximity zones
Now for the fun part - create your own proximity zones using `proximityObserver.zoneBuilder()`

```Kotlin
// Kotlin
val venueZone = proximityObserver.zoneBuilder()
                .forAttachmentKeyAndValue("venue", "office")
                .inFarRange()
                .withOnEnterAction{/* do something here */}
                .withOnExitAction{/* do something here */}
                .withOnChangeAction{/* do something here */}
                .create()
```

```Java
// Java
ProximityZone venueZone = 
    proximityObserver.zoneBuilder()
        .forAttachmentKeyAndValue("venue", "office")
        .inFarRange()
        .withOnEnterAction(new Function1<ProximityAttachment, Unit>() {
          @Override public Unit invoke(ProximityAttachment proximityAttachment) {
            /* Do something here */
            return null;
          }
        })
        .withOnExitAction(new Function1<ProximityAttachment, Unit>() {
              @Override
              public Unit invoke(ProximityAttachment proximityAttachment) {
                  /* Do something here */
                  return null;
              }
          })
        .withOnChangeAction(new Function1<List<? extends ProximityAttachment>, Unit>() {
          @Override
          public Unit invoke(List<? extends ProximityAttachment> proximityAttachments) {
            /* Do something here */
            return null;
          }
        })
        .create();
```
You zones can be defined with the below options: 
- **forAttachmentKeyAndValue** - the exact key and value that will trigger this zone actions. 
- **onEnterAction** - the action that will be triggered when the user enters the zone  
- **onExitAction** - the action that will be triggered when the user exits the zone.
- **onChangeAction** - triggers when there is a change in proximity attachments of a given key. If the zone consists of more than one beacon, this will help tracking the ones that are nearby inside the zone, while still remaining one `onEnter` and `onExit` event for the whole zone in general.  
- **inFarRange** - the far distance at which actions will be invoked. Notice that due to the nature of Bluetooth Low Energy, it is "desired" and not "exact." We are constantly improving the precision. 
- **inNearRange** - the near distance at which actions will be invoked.
- **inCustomRange** - custom desired trigger distance in meters. 

## 3. Start proximity observation
When you are done defining your zones, you will need to start the observation process:

```Kotlin
// Kotlin
val observationHandler = proximityObserver
               .addProximityZone(venueZone)
               .start()
```

```Java
// Java
ProximityObserver.Handler observationHandler =
       proximityObserver
           .addProximityZone(venueZone)
           .start();
```

The `ProximityObserver` will return `ProximityObserver.Handler` that you can use to stop scanning later. For example:
```Kotlin
// Kotlin
override fun onDestroy() {
    observationHandler.stop()
    super.onDestroy()
}
```

```Java
// Java
@Override
protected void onDestroy() {
    observationHandler.stop();
    super.onDestroy();
}
```

# Additional features

## Scanning for Estimote Telemetry
*Use case: Getting sensors data from your Estimote beacons.*

You can easily scan for raw `Estimote Telemetry` packets that contain your beacons' sensor data. All this data is broadcasted in the two separate sub-packets, called `frame A` and `frame B`. Our SDK allows you to scan for both of them separately, or to scan for the whole merged data at once (containing frame A and B data, and also the full device identifier).
Here is how to launch scanning for full telemetry data:

``` Kotlin
// KOTLIN
 bluetoothScanner = EstimoteBluetoothScannerFactory(applicationContext).getSimpleScanner()
        telemetryFullScanHandler =
                bluetoothScanner
                        .estimoteTelemetryFullScan()
                        .withOnPacketFoundAction {
                            Log.d("Full Telemetry", "Got Full Telemetry packet: $it") 
                        }
                        .withOnScanErrorAction { 
                            Log.e("Full Telemetry", "Full Full Telemetry scan failed: $it") 
                        }
                        .start()
```
You can use `telemetryFullScanHandler.stop()` to stop the scanning. Similarily to the `ProximityObserver` you can also start this scan in the foreground service using `getScannerWithForegroundService(notification)` method instead of `.getSimpleScanner()`.

Basic info about possible scanning modes:

`estimoteTelemetryFullScan()` - contains merged data from frame A and B, as well as full device id. Will be less frequently reported than individual frames.
`estimoteTelemetryFrameAScan()` - data from frame A + short device id. Reported on every new frame A.
`estimoteTelemetryFrameBScan()` - data from frame B + short device id. Reported on every new frame B.

> Tip: Read more about the Estimote Telemetry protocol specification [here](https://github.com/Estimote/estimote-specs/blob/master/estimote-telemetry.js). You can also check [our tutorial](http://developer.estimote.com/sensors/android-things/) about how to use the telemetry scanning on your Android Things device (RaspberryPi 3.0 for example).  

## Background scanning using foreground service
*Use case: Scanning when your app is in the background (not yet killed). Scanning attached to the notification object even when all activities are destroyed.*

It is now possible to scan when the app is in the background, but it needs to be handled properly according to the Android official guidelines. 

> IMPORTANT: Launching "silent bluetooth scan" without the knowledge of the user is not permitted by the system - if you do so, your service might be killed in any moment, without your permission. We don't want this behaviour, so we decided to only allow scanning in the background using a foreground service with a notification. You can implement your own solution, based on any kind of different service/API, but you must bear in mind, that the system might kill it if you don't handle it properly. 

1. Declare an notification object like this: 
``` Kotlin
// KOTLIN
val notification = Notification.Builder(this)
              .setSmallIcon(R.drawable.notification_icon_background)
              .setContentTitle("Beacon scan")
              .setContentText("Scan is running...")
              .setPriority(Notification.PRIORITY_HIGH)
              .build()
```

2. Use ` .withScannerInForegroundService(notification)` when building `ProximityObserver` via `ProximityObserverBuilder`:

3. To keep scanning active while the user is not in your activity (home button pressed) put start/stop in `onCreate()`/`onDestroy()` of your desired **ACTIVITY**. 

4. To scan even after the user has killed your activity (swipe in app manager) put start/stop in `onCreate()`/`onDestroy()` of your **CLASS EXTENDING APPLICATION CLASS**. 

> Tip: You can control the lifecycle of scanning by starting/stopping it in the different places of your app. If you happen to  never stop it, the underlying `foreground service` will keep running, and the notification will be still visible to the user. If you want such behaviour, remember to initialize the `notification` object correctly - add button to it that stops the service. Please, read more in the [official android documentation](https://developer.android.com/training/notify-user/build-notification.html) about managing notification objects. 

## Background scanning using Proximity Trigger (Android 8.0+)
*Use case: Displaying your notification when user enters the zone while having your app KILLED - the notification allows him to open your app (if you create it in such way). Triggering your `PendingIntent` when user enters the zone.*

Since Android version 8.0 there is a possibility to display a notification to the user when he enters the specified zone. This may allow him to open your app (by clicking the notification for example) that will start the proper foreground scanning.  
You can do this by using our `ProximityTrigger`, and here is how:

1. Declare an notification object like this: 
``` Kotlin
// KOTLIN
val notification = Notification.Builder(this)
              .setSmallIcon(R.drawable.notification_icon_background)
              .setContentTitle("Beacon scan")
              .setContentText("Scan is running...")
              // you can add here an action to open your app when user clicks the notification
              .setPriority(Notification.PRIORITY_HIGH)
              .build()
``` 
> Tip: Remember that on Android 8.0 you will also need to create a notification channel. [Read more here](https://developer.android.com/training/notify-user/build-notification.html).

2. Use `ProximityTriggerBuilder` to build `ProximityTrigger`:

``` Kotlin 
// KOTLIN
val triggerHandle = ProximityTriggerBuilder(applicationContext)
                .displayNotificationWhenInProximity(notification)
                .build()
                .start()
```
This will register the notification to be invoked when the user enters the zone of your beacons. You can use the `triggerHandle` to call `stop()` - this will deregister the system callback for you. 

Also, bear in mind, that the system callback **may be invoked many times**, thus displaying your notification again and again. In order to avoid this problem, you should add a button to your notification that will call `trigger.stop()` to stop the system scan. On the other hand, you can use `displayOnlyOnce()` method when building the `ProximityTrigger` object - this will fire your notification only once, and then you will need to call `start()` again.

> Known problems: The scan registraton gets cancelled when user disables bluetooth and WiFi on his phone. After that, the trigger may not work, and your app will need to be opened once again to reschedule the `ProximityTrigger`.

## Example app

To get a working prototype, check out the [example app](https://github.com/Estimote/Android-Proximity-SDK/tree/master/example/ProximityApp). It's a single screen app with three labels that change the background color when:

- you are in close proximity to the first desk,
- in close proximity to the second desk,
- when you are in the venue in general.

The demo requires at least two Proximity or Location beacons configured for Estimote Monitoring. It's enabled by default in dev kits shipped after mid-September 2017; to enable it on your own check out the [instructions](https://community.estimote.com/hc/en-us/articles/226144728-How-to-enable-Estimote-Monitoring-).

The demo expects beacons having specific attachments assigned:

- `venue`:`office` and `desk`:`mint` for the first one,
- `venue`:`office` and `desk`:`blueberry` for the second one.


## Documentation
Javadoc documentation available soon...

## Your feedback and questions
At Estimote we're massive believers in feedback! Here are some common ways to share your thoughts with us:
  - Posting issue/question/enhancement on our [issues page](https://github.com/Estimote/Android-Proximity-SDK/issues).
  - Asking our community managers on our [Estimote SDK for Android forum](https://forums.estimote.com/c/android-sdk).
  - Keep up with the development progress reports [in this thread on our forums](https://forums.estimote.com/t/changes-to-android-sdk-current-progress/7450). 

## Changelog
To see what has changed in recent versions of our SDK, see the [CHANGELOG](CHANGELOG.md).

