---
layout: post
title:
  "How to Migrate from StateNotifier to Notifier in Flutter Riverpod"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## Introduction

In my hobby application
[Mellotippet](https://github.com/molundb/mellotippet) I use
[riverpod](https://riverpod.dev/) for state management. I've been using
StateNotifiers but now the recommended approach is to use
Notifier/AsyncNotifier, ideally by generating the providers with the
@riverpod annotation. This migration guide shows how to do that.

The official migration guide can be found here:
[https://riverpod.dev/docs/migration/from_state_notifier](https://riverpod.dev/docs/migration/from_state_notifier)

## Migration Steps

I will migrate the SettingsController, which is the controller for the SettingsPage screen.

SettingsController looks like this:
```dart
part 'settings_controller.freezed.dart';

class SettingsController extends StateNotifier<SettingsControllerState> {
  SettingsController({required SettingsControllerState state}) : super(state);

  final AuthenticationRepository authRepository = getIt.get<AuthenticationRepository>();

  final MellotippetPackageInfo packageInfo = getIt.get<MellotippetPackageInfo>();

  static final provider =
      StateNotifierProvider<SettingsController, SettingsControllerState>(
    (ref) => SettingsController(
      state: const SettingsControllerState(appVersion: null),
    ),
  );

  ...
}

@freezed
class SettingsControllerState with _$SettingsControllerState {
  const factory SettingsControllerState({String? appVersion}) = _SettingsControllerState;
}
```

### Step 1: Add part, @riverpod, change extends and replace constructor with build()

We change this:
```dart
part 'settings_controller.freezed.dart';

class SettingsController extends StateNotifier<SettingsControllerState> {
  SettingsController({required SettingsControllerState state}) : super(state);

  ...
}
```

Into this:
```dart
part 'settings_controller.freezed.dart';
part 'settings_controller.g.dart';

@riverpod
class SettingsController extends _$SettingsController {
  @override
  SettingsControllerState build() => const SettingsControllerState(appVersion: null);

  ...
}
```

For the @riverpod annotation to work we need to import the [riverpod_annotation library](https://pub.dev/packages/riverpod_annotation). Run code generation and make sure there are no errors.

### Step 2: Remove the static provider variable

After running code generation the provider settingsControllerProvider is generated for us, which means we can remove our old static provider variable. SettingsController then looks like this: 
```dart
part 'settings_controller.freezed.dart';
part 'settings_controller.g.dart';

@riverpod
class SettingsController extends _$SettingsController {
  @override
  SettingsControllerState build() => const SettingsControllerState(appVersion: null);

  final AuthenticationRepository authRepository = getIt.get<AuthenticationRepository>();

  final MellotippetPackageInfo packageInfo = getIt.get<MellotippetPackageInfo>();

  ...
}

@freezed
class SettingsControllerState with _$SettingsControllerState {
  const factory SettingsControllerState({String? appVersion}) = _SettingsControllerState;
}
```

### Step 3: Use the generated provider in SettingsPage
Now we can replace the static provider variable with the generated provider.

We change this: 
```dart
class _SettingsPageState extends ConsumerState<SettingsPage> {
  SettingsController get controller =>
      ref.read(SettingsController.provider.notifier);

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(SettingsController.provider);
    
    ...
  }
}
```

Into this:
```dart
class _SettingsPageState extends ConsumerState<SettingsPage> {
  SettingsController get controller =>
      ref.read(settingsControllerProvider.notifier);

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(settingsControllerProvider);
    
    ...
  }
}
```



## Conclusion
And that's it! We've now migrated SettingsController to be generated into the correct type of provider with the @riverpod annotation. 

I hope this tutorial was helpful. The source code can be found here:
[https://github.com/molundb/mellotippet](https://github.com/molundb/mellotippet).
