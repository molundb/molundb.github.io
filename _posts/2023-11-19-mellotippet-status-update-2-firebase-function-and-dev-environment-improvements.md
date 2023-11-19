---
layout: post
title: "Mellotippet Status Update #2 - Firebase function and dev environment improvements"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## This was my plan for the week:

- Open source the project.
- Set proper Firebase Security Rules and unit test them.
- Make improvements to my firebase functions, like setting the region and making them idempotent.
- Make architectural improvements to the flutter code, like using freeze package for data classes.
- Get the design from my designer, and finally start implementing it. :)

## What I actually did this week

Again, I achieved a bit less than planned. What I ended up doing was:

- I open sourced the project. (Well, I made it public on <a href="https://github.com/molundb/mellotippet">github</a> at least).
- I added force upgrade functionality. (And published a blog post about it on <a href="https://martinlundberg.net/blog/2023/11/18/how-to-add-a-force-upgrade-dialog-in-flutter-with-firebase-remote-config">this page</a> and on <a href="https://medium.com/p/8a339fedda9e">Medium</a>.)
- I changed the region of my calculateScore firebase function and made it idempotent.
- I discussed the design with the desginer and got a first draft.

I had to spend most of the week working on my development environment and the force upgrade blog post. I learnt a ton about ruby, chruby, openssl, jekyll, liquid, scss, css and more. What probably took the most time was figuring out how to create beautiful code blocks, like this:

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

Here is a first draft of the design :)

![mellotippet design first draft]({{asset_path}}/mellotippet-design-first-draft.png){:width="800px"}

## My plan for next week:

- Make architectural improvements to the flutter code, like using freeze package for data classes.
- Set proper Firebase Security Rules and unit test them.
- Start looking into how to make a beautiful draggable UI element, which will be needed for the predictions page.