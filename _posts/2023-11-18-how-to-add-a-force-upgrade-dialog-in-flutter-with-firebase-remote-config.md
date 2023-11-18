---
layout: post
title: "How to Add a Force Upgrade Dialog in Flutter with Firebase Remote Config"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## Introduction

I recently added a Force Upgrade Dialog to my flutter app called Mellotippet, and I'd like to share why and how I did it. The source code can be found here: <a href="https://github.com/molundb/mellotippet">https://github.com/molundb/mellotippet</a>.

## Why Force Upgrade is important

There are several good reasons to have a force upgrade mechanism in place:

1. The backend can retire API versions and move faster, since they need to support fewer versions.
2. New security updates can be enforced.
3. If a severe bug is detected, it can be ensured that users upgrade to the hotfix version.

It's important to add force upgrade capabilities as early as possible, since you won't be able to force users before that point to upgrade their version.

In this blog post, we'll explore how to add a force upgrade dialog to a Flutter app using Firebase Remote Config. This approach allows you to prompt users to upgrade to the latest version directly from your app.

## Prerequisites

Before we begin, make sure you have the following set up:

1. A Flutter project.
2. A Firebase project with <a href="https://firebase.google.com/docs/remote-config">Remote Config</a> enabled.

### Step 1: Set up Firebase Remote Config

Define a Remote Config parameter for the minimum app version. In the Firebase console, add a parameter named `requiredMinimumVersion`. If you also want to be able to _recommend_ users to upgrade to a version without forcing them, add `recommendedMinimumVersion`.

![image tooltip]({{asset_path}}/firebase-remote-config-versions.png){:width="800px"}

### Step 2: Fetch values from Firebase Remote Config

1. **Install Dependencies:**

   Open your `pubspec.yaml` file and add a dependency on <a href="https://pub.dev/packages/firebase_remote_config">firebase_remote_config</a>.

    ```yaml
    dependencies:
      firebase_remote_config: ^X.X.X // Replace X.X.X with the latest version of the firebase_remote_config flutter package.
    ```

2. **Initialize Firebase:**

   In your `main.dart` file, initialize Firebase in the `main()` function:

    ```dart
    import 'package:firebase_core/firebase_core.dart';

    void main() async {
      WidgetsFlutterBinding.ensureInitialized();
      await Firebase.initializeApp();
      ...
    }
    ```

3. **Initialize Firebase Remote Config:**

    ```dart
    void main() async {
      WidgetsFlutterBinding.ensureInitialized();
      await Firebase.initializeApp();
      await MyFirebaseRemoteConfig.initialize();
      ...
    }
    ```

    ```dart
    import 'package:firebase_remote_config/firebase_remote_config.dart';

    class MyFirebaseRemoteConfig {
      final remoteConfig = FirebaseRemoteConfig.instance;

      static Future<void> initialize() async {
        final remoteConfig = FirebaseRemoteConfig.instance;
        await remoteConfig.setConfigSettings(
          RemoteConfigSettings(
            fetchTimeout: const Duration(minutes: 1),
            minimumFetchInterval: const Duration(hours: 1),
          ),
        );

        // These will be used before the values are fetched from Firebase Remote Config.
        await remoteConfig.setDefaults(const {
          'requiredMinimumVersion': '4.0.0',
          'recommendedMinimumVersion': '4.0.0',
        });

        // Fetch the values from Firebase Remote Config
        await remoteConfig.fetchAndActivate();

        // Optional: listen for and activate changes to the Firebase Remote Config values
        remoteConfig.onConfigUpdated.listen((event) async {
          await remoteConfig.activate();
        });
      }

      // Helper methods to simplify using the values in other parts of the code
      String getRequiredMinimumVersion() =>
        remoteConfig.getString('requiredMinimumVersion');

      String getRecommendedMinimumVersion() =>
        remoteConfig.getString('recommendedMinimumVersion');
    }
    ```

## Step 3: Get the user's current app version

To be able to know if the user has an outdated version, we need to know their current version. To achieve this, we'll use <a href="https://pub.dev/packages/package_info_plus">package_info_plus</a>.

1. **Install Dependencies:**

    Open your `pubspec.yaml` file and add a dependency on <a href="https://pub.dev/packages/package_info_plus">package_info_plus</a>.

    ```yaml
    dependencies:
      package_info_plus: ^X.X.X // Replace X.X.X with the latest version of the package_info_plus flutter package.
    ```

2. **Create a class to get and store the app's version:**

    ```dart
    import 'package:package_info_plus/package_info_plus.dart';

    class MellotippetPackageInfo {
      late String version;

      Future<void> initialize() async {
        PackageInfo packageInfo = await PackageInfo.fromPlatform();
        version = packageInfo.version;
      }
    }
    ```

3. **Intialize:**

    Call `initialize()` in `main.dart`. In this example we use <a href="https://pub.dev/packages/get_it">get_it</a> as a service locator, but you can use whatever you like. The important thing is that the version is fetched and can be used in the comparison in the next step.

    ```dart
    import 'package:firebase_core/firebase_core.dart';

    void main() async {
      WidgetsFlutterBinding.ensureInitialized();
      await Firebase.initializeApp();
      await getIt.get<MellotippetPackageInfo>().initialize();
      ...
    }
    ```

## Step 4: Define Force Upgrade Logic

Now that we have both the Remote Config version and the user's version, we can compare them to determine if we should show an upgrade dialog.

1. **Check if the dialog should be shown**

    Create a stateful widget called ForceUpgradePage that displays a dialog if a force upgrade is required.

    ```dart
    class ForceUpgradePage extends StatefulWidget {
      const ForceUpgradePage({super.key});

      @override
      State<ForceUpgradePage> createState() => _ForceUpgradeState();
    }

    class _ForceUpgradeState extends State<ForceUpgradePage> {

      // Get the necessary classes using get_it
      final packageInfo = getIt.get<MellotippetPackageInfo>();
      final featureFlagRepository = getIt.get<FeatureFlagRepository>();

      @override
      void initState() {
        super.initState();
        WidgetsBinding.instance.addPostFrameCallback((_) {
          // Get the current app version
          var appVersion = _getExtendedVersionNumber(packageInfo.version);

          // Get the required min version from Firebase Remote Config
          var requiredMinVersion = _getExtendedVersionNumber(
              featureFlagRepository.getRequiredMinimumVersion());

          // Get the recommended min version from Firebase Remote Config
          var recommendedMinVersion = _getExtendedVersionNumber(
              featureFlagRepository.getRecommendedMinimumVersion());

          // Compare the versions and display a dialog if the app version is lower than
          // the required or recommended version
          if (appVersion < requiredMinVersion) {
            _showUpdateVersionDialog(context, false);
          } else if (appVersion < recommendedMinVersion) {
            _showUpdateVersionDialog(context, true);
          } else {
            // If the current version is higher than the required and recommended version, navgiate to the next Page - in this case the LoginPage()
            Navigator.of(context).pushReplacement(
              MaterialPageRoute(
                builder: (context) => const LoginPage(),
              ),
            );
          }
        });
      }

      @override
      Widget build(BuildContext context) {
        return Scaffold(
          body: Container(),
        );
      }

      ...

      // Helper method to compare two semver versions.
      int _getExtendedVersionNumber(String version) {
        List versionCells = version.split('.');
        versionCells = versionCells.map((i) => int.parse(i)).toList();
        return versionCells[0] _ 100000 + versionCells[1] _ 1000 + versionCells[2];
      }
    }
    ```

2. **Display the force upgrade dialog:**

    Show the force upgrade dialog if the app version is lower than either the required or recommended min version.

    ```dart
    Future<void> _showUpdateVersionDialog(BuildContext context, bool isSkippable) async {
      return showDialog<void>(
        context: context,
        barrierDismissible: false,
        builder: (BuildContext context) {
          return AlertDialog(
            title: const Text("New version available"),
            content: SingleChildScrollView(
              child: ListBody(
              children: <Widget>[
                Text("Please update to the latest version of the app."),
              ],
            ),
          ),
          actions: <Widget>[
            // A "skip" button is only shown if it's a recommended upgrade
            isSkippable
              ? TextButton(
              child: const Text('Skip'),
                onPressed: () {
                Navigator.of(context).pushReplacement(
                  MaterialPageRoute(
                    builder: (context) => const LoginPage(),
                  ),
                );
                },
              )
              : Container(),
              TextButton(
                child: const Text('Update'),
                onPressed: () {
                  _launchAppOrPlayStore()
                },
            ),
          ],
          );
        },
      );
    }
    ```

## Step 5 (optional): Open App Store / Google Play Store

To make it easy for users to update their apps, make the "Update" button launch the App store on iOS devices and Google Play Store on Android devices.

1. **Install Dependencies:**

   Open your `pubspec.yaml` file and add a dependency on <a href="https://pub.dev/packages/url_launcher">url_launcher</a>.

    ```yaml
    dependencies:
      url_launcher: ^X.X.X // Replace X.X.X with the latest version of the url_launcher flutter package.
    ```

2. **Launch App/Play store**

    Launch the App store on iOS devices and Google Play Store on Android devices.

    ```dart
    void _launchAppOrPlayStore() {
        final appId = Platform.isAndroid ? 'YOUR_ANDROID_PACKAGE_ID' : 'YOUR_IOS_APP_ID';
        final url = Uri.parse(
          Platform.isAndroid
              ? "market://details?id=$appId"
              : "https://apps.apple.com/app/id$appId",
        );
        launchUrl(
          url,
          mode: LaunchMode.externalApplication,
        );
    }
    ```

## Step 6: Use ForceUpgradePage

Decide when you want to display the force upgrade dialog. Usually this is done as one of the first things in the app. In my case I set ForceUpgradeApp as the widget of the home parameter of my MaterialApp.

```
import 'package:flutter/material.dart';
import 'package:mellotippet/force_upgrade/force_upgrade_page.dart';

class MellotippetApp extends StatelessWidget {
const MellotippetApp({super.key});

@override
Widget build(BuildContext context) {
  return const MaterialApp(
    title: 'My title',
      home: const ForceUpgradePage(),
    );
  }
}
```

## Step 7: Run and test the app

Run and test the app manually on both iOS and Android. Try different values for `requiredMinimumVersion` and `recommendedMinimumVersion` to verify that everything works as expected.

*Required and recommended dialogs*

![image tooltip]({{asset_path}}/force-upgrade-dialog-required.png){:width="250px"}
![image tooltip]({{asset_path}}/force-upgrade-dialog-recommended.png){:width="250px"}


## Outro

I hope this tutorial is helpful. The source code can be found here: <a href="https://github.com/molundb/mellotippet">https://github.com/molundb/mellotippet</a>.
