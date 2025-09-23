---
title: Dart, Flutter, and the Case of the Missing Version Number
date: 2025-06-07 10:10:10 +0100
categories: [Work]
tags: [Flutter]
media_subpath: /assets/media/2025-06-07/
image:
  path: clint_eastwood.webp
---

## The SDK Shift
<b><i>Or what happens when you stop painting screens and start wiring them under the hood</i></b>


After many years as a mobile developer, switching to work on a mobile SDK has been… an interesting ride.
At first, it feels like I’ve crossed into backend territory — but truth be told, it’s still very much the same playground.
What do I mean?
Well, when you’re building a mobile app, chances are you’re mostly wrangling UI. Sure, some big projects have separate teams doing just business logic, never touching a single widget. But for most mobile devs, it’s UI all day, every day.

Mobile SDKs come in all shapes and sizes. Sometimes they deal with UI — sometimes, they’re mysterious little black boxes doing secret things under the hood.
That’s the case with the [Contentsquare SDK](https://contentsquare.com/). We don’t build any UI elements directly, but we do interact with them, just way deeper down in the stack.



## Flutter’s Runtime Riddle
<b><i>Or the mystery of the missing version number</i></b>


When you’re building an SDK for a framework like Flutter, there’s a good chance you’ll need to know which version of the framework the client is using. Why does it matter? Oh, let me count the ways:

#### 📊 Analytics
Clients often want analytics like app version, SDK version, and — you guessed it — framework version.

#### 🔌 API Compatibility
- Framework versions can introduce or remove APIs.
- Older versions may lack features, newer ones might break stuff.
- You’ll probably need to implement fallbacks.

#### 🛠 Client Support
When something breaks, clear version-aware logs help you (and them) debug faster.

#### 👮‍♂️ Security
- Some versions of the framework may have known vulnerabilities.
- Your SDK might want to warn or even bail out if it detects an insecure version.

#### 📈 Telemetry
- It’s super useful to know which framework versions your clients are running on.
- Helps you decide the minimum supported version based on actual usage, not gut feeling.

As you can see, knowing which framework version your client’s app is built with isn’t just nice to have — it’s survival gear for any SDK dev.


<h1> . . .</h1>


On Android and iOS, getting the OS version is easy-peasy:

```kotlin
val sdkVersion = android.os.Build.VERSION.SDK_INT
```

```swift
let systemVersion = UIDevice.current.systemVersion
```

Simple, right?

Now here’s the plot twist: Flutter doesn’t let you get the Flutter SDK version at <b>runtime</b>.
That’s right — no static variable, no system call, no magical `Platform.flutterVersion`. Nada.

And I wasn’t the only one hoping for a better way.
At some point, while deep in version-detective mode, I stumbled upon an old [GitHub PR](https://github.com/flutter/flutter/pull/140783) (from 2020) where someone proposed adding exactly this kind of feature — a static const with the current Flutter version.

I got my hopes up. Maybe, just maybe…

But alas, the dream was short-lived. The PR had already been closed for over a year. Rejected. Forgotten.

> It felt like discovering a treasure map — only to realize the “X” was just a typo ...


<h1> . . .</h1>


Yes, we still can pass the version at <b>build time</b>, like this:

```bash
flutter run --dart-define=FLUTTER_SDK_VERSION=$(flutter --version | head -n 1 | awk '{print $2}')
```

... and then read it in your app:

```dart
const String flutterVersion = String.fromEnvironment('FLUTTER_SDK_VERSION');
```

But here’s the catch: if you’re building an SDK, the last thing you want is to make clients jump through extra hoops. Asking them to modify their build process? That’s a big <b>“meh”</b> from us.
So this approach was off the table.



## A Glimmer of Hope
<b><i>Or the unexpected Dart-Flutter connection</i></b>


Turns out, the tunnel isn’t completely dark — there’s a hint of flashlight ahead: we can get the Dart SDK version at runtime:

```dart
import 'dart:io';

void main() {
  print(Platform.version);
}
```

Now you might be thinking: 
> <i>“How the hell does that help us?”</i> 😈

Well, it turns out there’s a pretty solid correlation between Dart and Flutter versions.
You can actually see it in [Flutter’s official archive](https://docs.flutter.dev/install/archive#stable-channel).

So, even though it’s not a one-to-one mapping, it’s close enough for practical use.
And that means… we can make a Dart-to-Flutter version map:

```dart
import 'package:pub_semver/pub_semver.dart';

final Map<Version, Version> dartToFlutterVersionMap = {
  Version.parse('3.8.1'): Version.parse('3.32.1'),
  Version.parse('3.8.0'): Version.parse('3.32.0'),
  Version.parse('3.7.2'): Version.parse('3.29.3'),
};
```

With this map, we can guesstimate the Flutter version from the Dart version at runtime — and that’s a huge win for SDK developers.



## Automating the Solution
<b><i>Or an introduction to a new package</i></b>


I’ll be honest — for a while, our approach was pretty old-school:
We kept a file with a manual `Dart→Flutter` version map.

Every time a new version dropped (every few weeks), we had to:

- update the file
- open a PR
- review it
- merge
- release
- repeat

Needless to say, that gets old fast. Not exactly what you’d call a scalable solution…

That’s when I decided to channel my inner lazy dev 🫠 — and turn the whole thing into a package:

Say hello to [dart_flutter_version](https://pub.dev/packages/dart_flutter_version) 🎉

What does it do? Pretty much the same thing as before, but a bit automated:

- A Dart script scrapes the official Flutter docs for version info
- It builds and updates the `Dart→Flutter` map
- A GitHub Action checks the docs every night
- If a new version shows up, the package auto-publishes an update to <b>pub.dev</b> 🚀

Now, SDKs (like ours) can depend on this package and grab the Flutter version at <b>runtime</b> — no extra work required from the client.



## Navigating Nuances
<b><i>Or understanding the package’s limitations</i></b>


Now, before you run off singing songs of automated glory, let’s pause for a dose of realism.

The `Dart→Flutter` version map is pretty reliable, but it’s not perfect.
Here are a few downsides of such an approach:

#### 🟡 Stable Channel Only

The map only covers stable Flutter releases. If you’re using `beta`, `dev`, or `master`, the version alignment could be quite different.

#### 🖥 Platform Bias

The mapping is based on macOS versions from the official docs. Things might differ on Windows or Linux (we see you, brave souls).

#### 🕐 Delay in Updates

There’s a nightly script checking for new versions, but it’s not <i>instant</i>. There could be a short delay between a new Flutter release and the package update.

#### 📉 Minimum Supported Version

Only Flutter 3.0.0 and above are supported. If you’re on an older version… well, it might be time for an upgrade anyway 😅

#### Bottom line:

The resolved version is <i>accurate enough</i> for most real-world use cases — but don’t bet your life (or a CI pipeline) on it being <i>pixel-perfect</i> in every scenario.



## The Auto-Update Advantage
<b><i>Or how to seamlessly stay current</i></b>


And now, the cherry on top: <b>auto-updates</b>. 🍒

Thanks to GitHub Actions, whenever a new `Dart→Flutter` version pair is detected, the package:

- updates the map
- bumps the <b>patch</b> version
- publishes to <b>pub.dev</b>

For example, here’s a snippet from the [changelog](https://pub.dev/packages/dart_flutter_version/changelog):

```md
1.0.5  
  Dart 3.8.1 → Flutter 3.32.1  

1.0.4  
  Dart 3.8.0 → Flutter 3.32.0
```

As long as SDK developers use the `^` symbol in their `pubspec.yaml`:

```bash
dart_flutter_version: ^1.0.5
```

they will automatically get the latest version the next time they run:

```bash
flutter pub get
```

This is especially useful in SDKs like ours, where clients might install the package today, next week, or in six months — and we want to make sure they are always getting the freshest version info, without lifting a finger 💅



## The Sequel Nobody Asked For
<b><i>Or how a comment brought a PR back from the grave</i></b>


Just when I thought the case was closed — literally — a little plot twist came along.

After publishing the shiny new `dart_flutter_version` package, I couldn’t help myself and dropped a message in that same old GitHub PR thread.
I included a link to the package and a bit of context, just for closure, really.

And then… something unexpected happened 😏

Within a month, the thread was reopened.
A few months later — boom 💥 — the PR was merged (May 21).
That long-awaited static variable? It’s finally becoming a reality.

This is a <b>huge win</b> for the Flutter community — and, ironically, the beginning of the end for my humble package 😅

But hey, it’ll still be useful for a while. Not everyone updates Flutter overnight — some apps take months (or years 😬) to catch up.
So until the dust settles and the new version becomes the norm:

> <b>Live long and prosper, `dart_flutter_version` 🖖</b>

## You did your duty, sir 🫡

![howdy](/howdy.gif)
