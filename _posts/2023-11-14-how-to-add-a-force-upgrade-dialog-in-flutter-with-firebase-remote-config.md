---
layout: post
title: "How to Add a Force Upgrade Dialog in Flutter with Firebase Remote Config"
date: 2023-11-14
---

### Why Force Upgrade is Important

There are several good reasons to have a force upgrade mechanism in place:

1. The backend can retire API versions and move faster, since they need to support fewer versions.
2. New security updates can be enforced.
3. If a severe bug is detected, it can be ensured that users upgrade to the hotfix version.

It's important to add force upgrade capabilities as early as possible, since you won't be able to force users before that point to upgrade their version.

In this blog post, we'll explore how to add a force upgrade dialog to a Flutter app using Firebase Remote Config. This approach allows you to prompt users to upgrade to the latest version directly from your app.

### Prerequisites

Before we begin, make sure you have the following set up:

1. A Flutter project.
2. A Firebase project with <a href="https://firebase.google.com/docs/remote-config">Remote Config</a> enabled.

### Step 1: Set Up Firebase Remote Config

Define a Remote Config parameter for the minimum app version. In the Firebase console, add a parameter named `required_minimum_version`. If you also want to be able to _recommend_ users to upgrade to a version without forcing them, add `recommended_minimum_version`.

### Step 2: Fetch values from Firebase Remote Config

1. **Install Dependencies:**
   Open your `pubspec.yaml` file and add the following dependency:

```yaml
dependencies:
  firebase_remote_config: ^X.X.X
```

asdasd
{: .text-orange-700 }

Replace `X.X.X` with the latest version of the `firebase_remote_config` package.

2. **Initialize Firebase:**
   In your `main.dart` file, initialize Firebase in the `main()` function:

````

import 'package:firebase_core/firebase_core.dart';

void main() async {
WidgetsFlutterBinding.ensureInitialized();
await Firebase.initializeApp();
...
}

```

3. **Configure Firebase Remote Config:**
   Initialize Firebase Remote Config:

```

void main() async {
WidgetsFlutterBinding.ensureInitialized();
await Firebase.initializeApp();
await MyFirebaseRemoteConfig.initialize();
...
}

```

```

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

    await remoteConfig.setDefaults(const {
      'requiredMinimumVersion': '4.0.0',
      'recommendedMinimumVersion': '4.0.0',
    });

    await remoteConfig.fetchAndActivate();

    remoteConfig.onConfigUpdated.listen((event) async {
      await remoteConfig.activate();
    });

}

String getRequiredMinimumVersion() =>
remoteConfig.getString('requiredMinimumVersion');

String getRecommendedMinimumVersion() =>
remoteConfig.getString('recommendedMinimumVersion');
}

```

The methods `getRequiredMinimumVersion()` and `getRecommendedMinimumVersion()` exist to simplify using the values in other parts of the code.

### Step 3: Get the user's current version of the app

To be able to know if the user has an outdated version, we need to know the user's current version. To achieve this, use <a href="https://pub.dev/packages/package_info_plus">package_info_plus</a>.

1. Open your `pubspec.yaml` file and add a dependency on package_info_plus https://pub.dev/packages/package_info_plus.

```

dependencies:
package_info_plus: ^X.X.X

```

Replace `X.X.X` with the latest version of the `package_info_plus` package.

2. Create a class to get and store the app's version.

```

import 'package:package_info_plus/package_info_plus.dart';

class MellotippetPackageInfo {
late String version;

Future<void> initialize() async {
PackageInfo packageInfo = await PackageInfo.fromPlatform();
version = packageInfo.version;
}
}

```

3. Call `initialize()` in `main.dart`. In this example we use <a href="https://pub.dev/packages/get_it">get_it</a> as a service locator, but you can use whatever you like. The important thing is that the version is fetched and can be used in the comparison in the next step.

```

import 'package:firebase_core/firebase_core.dart';

void main() async {
WidgetsFlutterBinding.ensureInitialized();
await Firebase.initializeApp();
await getIt.get<MellotippetPackageInfo>().initialize();
...
}

```

### Step 4: Define Force Upgrade Logic

Now that we have both the Remote Config version and the user's version, we can compare them to determine if we should show an upgrade dialog.

1. **Create a widget to check for Force Upgrade:**
   Create a stateful widget called ForceUpgradePage that checks if a force upgrade is required, and displays a dialog.

```

class ForceUpgradePage extends StatefulWidget {
const ForceUpgradePage({super.key});

@override
State<ForceUpgradePage> createState() => \_ForceUpgradeState();
}

class \_ForceUpgradeState extends State<ForceUpgradePage> {
final packageInfo = getIt.get<MellotippetPackageInfo>();
final myFirebaseRemoteConfig = getIt.get<MyFirebaseRemoteConfig>();

@override
void initState() {
super.initState();
WidgetsBinding.instance.addPostFrameCallback((\_) {
var appVersion = \_getExtendedVersionNumber(packageInfo.version);
var requiredMinVersion = \_getExtendedVersionNumber(
myFirebaseRemoteConfig.getRequiredMinimumVersion());
var recommendedMinVersion = \_getExtendedVersionNumber(
myFirebaseRemoteConfig.getRecommendedMinimumVersion());

      if (appVersion < requiredMinVersion) {
        _showUpdateVersionDialog(context, false);
      } else if (appVersion < recommendedMinVersion) {
        _showUpdateVersionDialog(context, true);
      } else {
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

```

In `initState()` the versions are compared and a dialog is shown if needed. If not, the user is navigated to the next page, which in this case is `LoginPage()`.

The method `_getExtendedVersionNumber()` is used to compare two <a href="https://semver.org/">semver</a> versions.

```

int \_getExtendedVersionNumber(String version) {
List versionCells = version.split('.');
versionCells = versionCells.map((i) => int.parse(i)).toList();
return versionCells[0] _ 100000 + versionCells[1] _ 1000 + versionCells[2];
}

```

The method `_showUpdateVersionDialog()` displays an alert dialog with information text and a button that opens the app in the App store on iOS and Google Play Store on Android. If the version is only recommended there is also a button to skip. If desired you can add logic to not show the dialog anymore if skip is pressed by using <a href="https://pub.dev/packages/shared_preferences">shared_preferences</a>.

```

Future<void> \_showUpdateVersionDialog(BuildContext context, bool isSkippable) async {
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
// TODO: Open app in app store / google play store
},
),
],
);
},
);
}
}

int \_getExtendedVersionNumber(String version) {
List versionCells = version.split('.');
versionCells = versionCells.map((i) => int.parse(i)).toList();
return versionCells[0] _ 100000 + versionCells[1] _ 1000 + versionCells[2];
}

```

The package <a href="https://pub.dev/packages/url_launcher">url_launcher</a> is used to open the links.

### Step 5: Open App Store / Google Play Store

To make it easy for the user to update their app, make the "Update" button open the App store on iOS devices and Google Play Store on Android devices.

```

TextButton(
child: const Text('Update'),
onPressed: () {
final appId = Platform.isAndroid
? 'YOUR_ANDROID_PACKAGE_ID'
: 'YOUR_IOS_APP_ID';

    	final url = Uri.parse(
    	  Platform.isAndroid
    	      ? "https://play.google.com/store/apps/details?id=$appId"
    	      : "https://apps.apple.com/app/id$appId",
    },

),

```

### Step 6: Use ForceUpgradePage

Set ForceUpgradeApp as the widget of the home parameter of your MaterialApp.

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

### Step 7: Run and test the app

Run and test the app manually on both iOS and Android to verify that it works. Try different values for requiredMinimumVersion and recommendedMinimumVersion.

### Outro

This is the tutorial I wish I had when implementing this myself. I hope it can be helpful to someone. I'll post a link to Mellotippet project as soon as I've made it public.
```
````
