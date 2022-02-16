# Using Flutter flavors to separate the DEV and LIVE environment


These are the requirements for our app:

* Our Flutter app should target iOS and Android.
* We want a DEV version and a LIVE version of our app, each targeting a different API URL.
* Developers should never have to manually change any code to switch between the environments.
* We want to be able to have the DEV app and the LIVE app installed on the same device at the same time.

The best way to solve these requirements in Flutter is to use __flavors__. There are also some other tutorials for this linked on the official [Flutter docs](https://flutter.dev/docs/deployment/flavors) which might be helpful.

The [code for this guide](https://github.com/cwe1ss/flutter-flavors-ci-cd) is stored on GitHub and changes for each section are separate commits that are linked in the section below.

<!--more-->

As we need to change settings in XCode, we need a Mac with Android Studio and XCode for this tutorial.

You also need to have Flutter installed already. Follow the [Getting started-docs](https://flutter.dev/docs/get-started/install) if you still need to install it.

I'm using `Flutter v1.22.0` for this tutorial.

## Create the Git repository and clone it locally

Create a new Git repository. Mine is: [https://github.com/cwe1ss/flutter-flavors-ci-cd](https://github.com/cwe1ss/flutter-flavors-ci-cd)

```cmd
git clone https://github.com/cwe1ss/flutter-flavors-ci-cd.git
cd flutter-flavors-ci-cd/
```

## Create the Flutter app

Let's get started by creating the Flutter app, named `flutter_flavors` via the Flutter CLI directly in the root folder of our repository:

```cmd
flutter create --project-name flutter_flavors .
```

Run the app on an Android device/emulator to ensure it works.

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/flutter-flavors-ci-cd/commit/0819ce8b51ddc39d8b7aebf71aea4f2da36a187c)

## Add a Flutter build configuration for each flavor in Android Studio

We want to have two flavors called `dev` and `live`.

If you want to launch a flutter app with a flavor, you have to use the `--flavor NAME` parameter in the *Flutter CLI*. To automatically start the app with a flavor in *Android Studio* we need to change the build configurations:

* Find `main.dart` in the Android Studio top toolbar and select `Edit Configurations...`. This opens the "Run/Debug Configurations" window.
* Change the `Name:` field to `dev`
* For `Build flavor:` set `dev` as well.
* Make sure "Share through VCS" is selected.
* Copy the dev configuration (It's an icon in the top left of the window)
* Change the `Name:` and `Build flavor:` values to `live`
* Make sure "Share through VCS" is selected as well
* Close the dialog. Instead of "main.dart", it will now display "dev" in the top toolbar.

*IMPORTANT: Flavor names may not start with "test" as that's not allowed by Android.*

### Add the build configurations to Git

When you select "Share through VCS", Android studio will create files in the `.idea/runConfigurations` folder, however they'll be ignored by the existing .gitignore file.

We'll therefore add these files manually to Git, so that other users in the team can use it as well:

```cmd
git add .idea/runConfigurations/dev.xml -f
git add .idea/runConfigurations/live.xml -f
git commit -m 'Persist flutter build configurations'
```

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/flutter-flavors-ci-cd/commit/36e4eacccf68eb131cd1aa6ab1ac025df56aeb05)

## Set up flavors for Android

In order to actually use different flavors, we need to set them up in the `lib`-folder and in each platform (`android`, `ios`).

We'll start with the `android` part.

### Add the method channel in Android code

When the app starts, Flutter needs a way to ask the native platform which flavor it has been started with. To communicate with native code, Flutter uses `method channels`.

Go to `android/app/src/main/kotlin/com.example.flutter_flavors` and replace everything except the first line (the package import) with the following code. This will set up the method channel that returns the `BuildConfig.FLAVOR` value, which is a built-in value of Android.

```kotlin
import androidx.annotation.NonNull;
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugins.GeneratedPluginRegistrant

class MainActivity: FlutterActivity() {
    override fun configureFlutterEngine(@NonNull flutterEngine: FlutterEngine) {
        GeneratedPluginRegistrant.registerWith(flutterEngine);

        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "flavor").setMethodCallHandler {
            call, result -> result.success(BuildConfig.FLAVOR)
        }
    }
}
```

### Add the flavor-settings to the Android build config

In Android, the native flavor-specific values are stored in `android/app/src/build.gradle` via the `android.flavorDimensions` and `android.productFlavors` keys.

We'll use these keys to set up the flavor-specific `applicationId` and the flavor-specific display name for the app. This is important because we want to be able to have both flavors of the app installed at the same time.

The `applicationId` is the unique app id for each flavor in the Google Play store. Once deployed to Google Play, this can not be changed anymore!


Therefore, add the following two things within the `android { ... }` section:

```gradle
android {
    // ... all existing things like `sourceSets`, ...

    flavorDimensions "app"

    productFlavors {
        dev {
            dimension "app"
            applicationId "at.chwe.flutterflavors.dev"
            resValue "string", "app_name", "DEV Flutter Flavors"
        }
        live {
            dimension "app"
            applicationId "at.chwe.flutterflavors"
            resValue "string", "app_name", "Flutter Flavors"
        }
    }
}
```

### Use the app_name in the AndroidManifest.xml

The `applicationId` is a well-known key that will already be used when the app is launched with a given flavor.

However, we'll need to do some more work to get the app_name working: Open `android/app/src/main/AndroidManifest.xml` and replace the `<application android:label="flutter_flavors" />` key with `<application android:label="@string/app_name" />`.

### Run the app again on Android

As we're now using new applicationIds, make sure the existing "flutter_flavors"-app is uninstalled from your device.

Now, launch the app in Android Studio with the "dev" build configuration.

Close the app in the device and check your application list, the app name will now display "DEV Flutter Flavors"!

Stop the app in Android Studio, change the build configuration to "live" and launch the app again.

You'll now have both flavors of your app installed on your Android device!

![Both flavors are installed on android](/images/posts/2020/flutter_flavors_app_icons_on_android.png)

Our native Android configuration is now complete.

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/flutter-flavors-ci-cd/commit/099284216215e0a8578518399117d8e28db31b75)

## Get the flavor in our Flutter code

As described in our requirements, we want to target different API endpoints per flavor so we need a way to get the current flavor in our Flutter code.

We'll first add a class called `FlavorSettings` in a new file called `lib/flavor_settings.dart` that will hold all of our flavor-specific settings that we only need in our Flutter code:

```dart
/// Contains the hard-coded settings per flavor.
class FlavorSettings {
  final String apiBaseUrl;
  // TODO Add any additional flavor-specific settings here.

  FlavorSettings.dev()
    : apiBaseUrl = 'https://dev.flutter-flavors.chwe.at';

  FlavorSettings.live()
    : apiBaseUrl = 'https://flutter-flavors.chwe.at';
}
```
Next, we'll use this in `main.dart`, where we'll read the flavor via our method channel from the native platform and we'll create the corresponding `FlavorSettings`-object. We'll also have to make our `main`-method async for that.

When done, your `main.dart` should contain the following code before the `class MyApp extends StatelessWidget {` line:

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

import 'flavor_settings.dart';

Future<void> main() async {
  // NOTE: This is required for accessing the method channel before runApp().
  WidgetsFlutterBinding.ensureInitialized();

  final settings = await _getFlavorSettings();
  print('API URL ${settings.apiBaseUrl}');

  runApp(MyApp());
}

Future<FlavorSettings> _getFlavorSettings() async {
  String flavor = await const MethodChannel('flavor')
        .invokeMethod<String>('getFlavor');

  print('STARTED WITH FLAVOR $flavor');

  if (flavor == 'dev') {
    return FlavorSettings.dev();
  } else if (flavor == 'live') {
    return FlavorSettings.live();
  } else {
    throw Exception("Unknown flavor: $flavor");
  }
}

// ... class MyApp extends StatelessWidget {
```

Run the app with `dev` build configuration and look at the console output. It will display the following statements:

```
I/flutter ( 4458): STARTED WITH FLAVOR dev
I/flutter ( 4458): API URL https://dev.flutter-flavors.chwe.at
```

Switch to the `live` build configuration and run your app again. This time, the console will display the following statements:

```
I/flutter ( 4615): STARTED WITH FLAVOR live
I/flutter ( 4615): API URL https://flutter-flavors.chwe.at
```

That's it! We can now access the current flavor from within Flutter and we can have flavor-specific settings.

You can pass the `FlavorSettings`-instance down to your widgets manually, or you can use e.g. the `provider`-package to access it via dependency injection in your widgets.

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/flutter-flavors-ci-cd/commit/64ee8642551702dd4d36f15c5f788befe8c6bc9a)

## Set up flavors for iOS

Unfortunately, setting up flavors in iOS is more complex and we'll have to use XCode and its UI for most of the steps.

Let's try building our app with a flavor for iOS now to see, what kind of error we get:

```
flutter build ios --flavor dev

The Xcode project does not define custom schemes. You cannot use the --flavor option.
```

This means, that on iOS we have to rely on a feature called "custom schemes" to represent our flutter flavors. Setting them up requires multiple steps.

### Create custom build configurations for the flavors

Let's open our `ios`-folder in XCode and start by creating our custom build configurations:

* Make sure the root "Runner" node is selected in XCode
* In the main window, select the "Runner" node below "PROJECT" (NOT below TARGETS)
* Select the "Info" tab
* In the "Configurations" section, do the following:
  * Rename "Debug" to "Debug-dev"
  * Rename "Release" to "Release-dev"
  * Rename "Profile" to "Profile-dev"
  * Duplicate "Debug-dev" and rename it to "Debug-live"
  * Duplicate "Release-dev" and rename it to "Release-live"
  * Duplicate "Profile-dev" and rename it to "Profile-live"

This means, for every flavor, we need a separate "Debug", "Release" & "Profile" configuration.

![Configurations in XCode](/images/posts/2020/flutter_flavors_ios_configurations.png)

### Assign build configurations to custom schemes

Now we can set up the actual "custom schemes" by doing the following:

* Make sure the root "Runner" node is selected in XCode
* Select "Product -> Scheme -> Manage Schemes..." in the main toolbar.
* To get the "dev" scheme:
  * Select the "Runner" scheme, click on the settings-icon in the top left and select "Duplicate"
  * Rename the scheme to "dev"
  * Make sure "Shared" is selected
  * Close the dialog
* To get the "live" scheme:
  * Select the "Runner" scheme again, click on the settings-icon in the top left and select "Duplicate"
  * Rename the scheme to "live"
  * For each of the sections ("Run", "Test", "Profile", "Analyze", "Archive") on the left, change the build configuration to the corresponding "-live" version.
  * Make sure "Shared" is selected
  * Close the dialog

![live scheme in XCode](/images/posts/2020/flutter_flavors_ios_live_scheme.png)

Back in the "schemes" list, you can now delete the existing "Runner" scheme. This should result in the list looking like this:

![scheme list in XCode](/images/posts/2020/flutter_flavors_ios_schemes.png)

### Adding the method channel for iOS

Building the app now shoud succeed, however when you try to run it, it will fail with the following error:

```
[VERBOSE-2:ui_dart_state.cc(177)] Unhandled Exception: MissingPluginException(No implementation found for method getFlavor on channel flavor)
```

That's because we haven't yet implemented the method channel that Flutter uses to get the current flavor from the native platform.

To add this, we need to add some code to the `application()`-function of the `Runner/AppDelegate.swift`-file in XCode. The finished file should look like this:

```swift
import UIKit
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    GeneratedPluginRegistrant.register(with: self)

    let controller = window.rootViewController as! FlutterViewController

    let flavorChannel = FlutterMethodChannel(
        name: "flavor",
        binaryMessenger: controller.binaryMessenger)

    flavorChannel.setMethodCallHandler({(call: FlutterMethodCall, result: @escaping FlutterResult) -> Void in
        // Note: this method is invoked on the UI thread
        let flavor = Bundle.main.infoDictionary?["App - Flavor"]
        result(flavor)
    })

    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

This will set up a method channel handler that reads the current flavor from a `Bundle.main.infoDictionary` with a key called `App - Flavor`.

### Set up the flavor value per scheme

The `Bundle.main.infoDictionary` from before refers to the `Runner/Info.plist` file and `App - Flavor` is a custom key that we have to add there manually next.

So open the `Runner/Info.plist` file in XCode and and add a new row with the following settings:
* Key: `App - Flavor`
* Type: `String`
* Value `$(APP_FLAVOR)`

We now have the key but we still don't have the actual flavor-specific values per scheme. To add them, we now have to do the following:

* Select the root "Runner" node in your XCode project structure
* Select "Runner" below __TARGETS__
* Select the "Build settings" tab
* Click on the + to add a new User-defined setting
* Name it `APP_FLAVOR`
* Expand the node by clicking on the little arrow on the left of the row and add the actual flavor value to each build configuration:
  * Debug-dev: `dev`
  * Debug-live: `live`
  * Profile-dev: `dev`
  * Profile-live: `live`
  * Release-dev: `dev`
  * Release-live: `live`

When done, it should look like this:
![flavor setting in XCode](/images/posts/2020/flutter_flavors_ios_flavor_setting.png)

### Run the iOS app

We should now be able to select the "dev"-scheme in the top navigation bar of XCode and run the app.

*NOTE: If you get weird build errors from XCode, try switching between the dev/live schemes or try restarting XCode or running the iOS app from Android Studio.*

You should now see similar console output like for the Android app:

```
2020-10-03 14:44:05.525493+0200 Runner[26055:336596] flutter: STARTED WITH FLAVOR dev
2020-10-03 14:44:05.526672+0200 Runner[26055:336596] flutter: API URL https://dev.flutter-flavors.chwe.at
```

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/flutter-flavors-ci-cd/commit/608246ea5f948f2a63dc1d7450a77468428a4fc4)

Great, we now have set up our flavors for iOS as well!

## Set the bundle id and app name per flavor for iOS

You might have noticed that the app name on the iPhone still is "flutter_flavors". Also, when you run both flavors, you still only have one app on your phone:

![wrong app name on iPhone](/images/posts/2020/flutter_flavors_ios_wrong_appname.png)

Remember, that for Android we've set those values in the `build.gradle` file.

To make things flavor-specific in iOS, we need to do something similar like we've done for the flavor value itself, where we've configured a key in `Info.plist` and then set different values in the "TARGETS/Runner -> Build Settings" tab.

### Set the flavor-specific bundle identifier

The `Info.plist` file already contains a key named `Bundle identifier` that already contains a dynamic value `$(PRODUCT_BUNDLE_IDENTIFIER)`, so we don't have to create another entry in this file.

Instead, we just have to modify this key in the the build settings:
* In XCode, select the root "Runner" node in the project explorer
* Select "Runner" below __TARGETS__
* Go to the "Build Settings" tab
* In the "Packaging" section, find the "Product Bundle Identifier" key
* Expand the key by clicking on the small arrow on the left
* Set the value per build configuration:
  * Debug-dev: `at.chwe.flutterflavors.dev`
  * Debug-live: `at.chwe.flutterflavors`
  * Profile-dev: `at.chwe.flutterflavors.dev`
  * Profile-live: `at.chwe.flutterflavors`
  * Release-dev: `at.chwe.flutterflavors.dev`
  * Release-live: `at.chwe.flutterflavors`

![bundle id per ios config](/images/posts/2020/flutter_flavors_ios_bundle_id.png)

### Set the app name

To have separate display names per flavor, do the following:
* In XCode, select the root "Runner" node in the project explorer
* Select "Runner" below __TARGETS__
* Select the "Info" tab
* Change the value of the key `Bundle name` to `$(APP_NAME)`.
* Go to the "Build Settings" tab
* Add a new User-Defined setting
* Name it `APP_NAME`
* Expand the APP_NAME-node by clicking on the small arrow on the left side of the node.
* Set the value per build configuration:
  * Debug-dev: `DEV Flutter Flavors`
  * Debug-live: `Flutter Flavors`
  * Profile-dev: `DEV Flutter Flavors`
  * Profile-live: `Flutter Flavors`
  * Release-dev: `DEV Flutter Flavors`
  * Release-live: `Flutter Flavors`

![app name per ios config](/images/posts/2020/flutter_flavors_ios_app_name.png)

### Run the app again with the dev and live flavors

Delete the existing "flutter_flavors" app from your iPhone and run it again with each flavor. You should now have both apps with the correct name on your phone:

![app icons on the iPhone](/images/posts/2020/flutter_flavors_ios_app_icons.png)

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/flutter-flavors-ci-cd/commit/6b4ead183f8b3f9128fa2b4d24b277056b747909)

## Add your own flavor-specific settings

If you need another flavor-specific setting, you have to know if it is a platform-specific setting, that needs to be integrated directly into the `android` and `ios` folders or if it is a setting that is only required in your Flutter code.

For platform-specific settings, the above guides for setting the app name and bundle id should help.

For settings that you only need in your Flutter-code, just add them to the `FlavorSettings`-class that we've created above.

## Summary

We've now set up our Flutter project to have multiple flavors. We use those flavors to separate our app environments (DEV & LIVE). This way we don't need to e.g. manually comment out code to switch our API URL or any other settings. We can also have both versions installed side by side, which makes development and support much easier as we can develop on the DEV version while we still can use the LIVE version which is deployed to the stores.

