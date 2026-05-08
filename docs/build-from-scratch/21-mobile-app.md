# Part 21 — Mobile App Wiring & Branding

> Goal of this part: a Flutter app that signs in via Firebase, opens
> the listen WebSocket, displays live transcripts, shows conversations
> + memories + chat, and looks like *your* product.

This part is intentionally short relative to its size — the Flutter
ecosystem is huge, and every screen is a sub-project. Treat this as
the wiring guide; pick UI ergonomics yourself.

## 1. Create the project

```bash
cd <<YOUR_BRAND>>
flutter create --org me.<<YOUR_BRAND>> \
  --platforms=ios,android \
  --description "<<YOUR_BRAND>> — your second brain" \
  --project-name <<your_brand>>_app \
  app
```

(Flutter project names are snake_case and must start with a letter.)

`cd app && flutter run` should boot the demo counter app.

## 2. Brand the shell

### App name

Edit `app/android/app/src/main/AndroidManifest.xml`:

```xml
<application
    android:label="<<YOUR_BRAND>>"
    ...>
```

Edit `app/ios/Runner/Info.plist`:

```xml
<key>CFBundleDisplayName</key>
<string><<YOUR_BRAND>></string>
```

### Bundle identifiers

Already set when you ran `flutter create --org me.<<YOUR_BRAND>>`,
but verify:

- iOS: `app/ios/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER`
- Android: `app/android/app/build.gradle` → `applicationId`

Both should be `me.<<YOUR_BRAND>>.app` (or whatever you decided in
Part 01).

### App icons & splash

Use [`flutter_launcher_icons`](https://pub.dev/packages/flutter_launcher_icons)
and [`flutter_native_splash`](https://pub.dev/packages/flutter_native_splash):

```yaml
# in app/pubspec.yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.1
  flutter_native_splash: ^2.4.0

flutter_launcher_icons:
  android: true
  ios: true
  image_path: "../branding/<<YOUR_BRAND>>/mobile/icon.png"
  background_color_ios: "#FFFFFF"

flutter_native_splash:
  color: "#FFFFFF"
  image: "../branding/<<YOUR_BRAND>>/mobile/splash.png"
```

```bash
cd app
flutter pub get
dart run flutter_launcher_icons
dart run flutter_native_splash:create
```

## 3. Required dependencies

```yaml
# app/pubspec.yaml — under dependencies:
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  google_sign_in: ^6.2.1
  sign_in_with_apple: ^6.1.0
  firebase_messaging: ^15.1.3

  http: ^1.2.2
  web_socket_channel: ^3.0.1

  flutter_blue_plus: ^1.32.13           # BLE
  permission_handler: ^11.3.1
  audioplayers: ^6.1.0
  record: ^5.1.2                        # phone mic fallback
  opus_dart: ^3.0.0                     # decode/encode opus on the phone

  go_router: ^14.6.1                    # navigation
  riverpod: ^2.5.1                      # state management
  flutter_riverpod: ^2.5.1
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0
  intl: ^0.19.0

dev_dependencies:
  build_runner: ^2.4.13
  freezed: ^2.5.7
  json_serializable: ^6.8.0
```

```bash
flutter pub get
```

## 4. Firebase setup for the app

```bash
cd app
flutterfire configure --project=<<YOUR_GCP_PROJECT_ID>>
```

This generates `lib/firebase_options.dart` and configures iOS +
Android. Enable iOS push notifications in the Firebase Console
(Cloud Messaging → APNs key).

## 5. Project structure

```
app/lib/
  main.dart                     ← entry, app shell
  firebase_options.dart         ← generated
  config.dart                   ← BASE_API_URL, env flags
  features/
    auth/        → sign_in_screen.dart, auth_repository.dart
    onboarding/  → consent_screen.dart, language_screen.dart, profile_screen.dart
    listen/      → listen_screen.dart, ble_service.dart, audio_streamer.dart
    conversations/ → list_screen.dart, detail_screen.dart, segment_tile.dart
    memories/    → list_screen.dart, detail_screen.dart
    chat/        → chat_screen.dart, message_bubble.dart
    devices/     → pair_screen.dart, scan_screen.dart
    settings/    → settings_screen.dart, integrations_screen.dart
  services/      → api_client.dart, listen_ws.dart, fcm_service.dart
  models/        → user.dart, conversation.dart, segment.dart, memory.dart, action_item.dart
  common/        → app_theme.dart, widgets/ (loading, error, empty)
  l10n/          → arb files; flutter gen-l10n produces AppLocalizations
```

## 6. The API client

```dart
// in app/lib/services/api_client.dart
import 'dart:convert';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:http/http.dart' as http;

class ApiClient {
  ApiClient({required this.baseUrl});

  final String baseUrl;
  final _client = http.Client();

  Future<Map<String, String>> _headers() async {
    final user = FirebaseAuth.instance.currentUser;
    if (user == null) throw StateError('Not signed in');
    final token = await user.getIdToken();
    return {
      'Authorization': 'Bearer $token',
      'Content-Type': 'application/json',
    };
  }

  Future<Map<String, dynamic>> get(String path) async {
    final r = await _client.get(Uri.parse('$baseUrl$path'), headers: await _headers());
    if (r.statusCode >= 300) throw HttpException(r.statusCode, r.body);
    return jsonDecode(r.body) as Map<String, dynamic>;
  }

  Future<Map<String, dynamic>> post(String path, Map<String, dynamic> body) async {
    final r = await _client.post(Uri.parse('$baseUrl$path'),
        headers: await _headers(), body: jsonEncode(body));
    if (r.statusCode >= 300) throw HttpException(r.statusCode, r.body);
    return jsonDecode(r.body) as Map<String, dynamic>;
  }

  void close() => _client.close();
}

class HttpException implements Exception {
  HttpException(this.code, this.body);
  final int code;
  final String body;
  @override
  String toString() => 'HttpException($code): $body';
}
```

## 7. The listen WebSocket

```dart
// in app/lib/services/listen_ws.dart
import 'dart:async';
import 'dart:convert';
import 'dart:typed_data';

import 'package:firebase_auth/firebase_auth.dart';
import 'package:web_socket_channel/io.dart';
import 'package:web_socket_channel/web_socket_channel.dart';

class ListenWebSocket {
  ListenWebSocket({required this.baseUrl});

  final String baseUrl;
  WebSocketChannel? _ch;
  final _events = StreamController<Map<String, dynamic>>.broadcast();
  Stream<Map<String, dynamic>> get events => _events.stream;

  Future<void> connect({String codec = 'pcm16', int sampleRate = 16000, String language = 'en'}) async {
    final token = await FirebaseAuth.instance.currentUser!.getIdToken();
    final uri = Uri.parse('${baseUrl.replaceFirst('https', 'wss')}/v1/listen'
        '?token=$token&codec=$codec&sample_rate=$sampleRate&language=$language');
    _ch = IOWebSocketChannel.connect(uri);
    _ch!.stream.listen((m) {
      if (m is String) _events.add(jsonDecode(m) as Map<String, dynamic>);
    }, onDone: () => _events.add({'type': 'closed'}),
       onError: (e) => _events.add({'type': 'error', 'message': '$e'}));
  }

  void sendAudio(Uint8List frame) => _ch?.sink.add(frame);

  Future<void> close() async {
    await _ch?.sink.close();
    _ch = null;
  }
}
```

## 8. Sign-in flow (the only non-trivial part)

```dart
// in app/lib/features/auth/auth_repository.dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';

class AuthRepository {
  Future<UserCredential> signInWithGoogle() async {
    final account = await GoogleSignIn().signIn();
    if (account == null) throw StateError('cancelled');
    final auth = await account.authentication;
    final cred = GoogleAuthProvider.credential(accessToken: auth.accessToken, idToken: auth.idToken);
    return FirebaseAuth.instance.signInWithCredential(cred);
  }

  Future<UserCredential> signInWithApple() async {
    final apple = await SignInWithApple.getAppleIDCredential(scopes: [
      AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName,
    ]);
    final cred = OAuthProvider('apple.com').credential(
      idToken: apple.identityToken, rawNonce: null,
    );
    return FirebaseAuth.instance.signInWithCredential(cred);
  }

  Future<void> signOut() => FirebaseAuth.instance.signOut();
}
```

## 9. Branding the look

`app/lib/common/app_theme.dart`:

```dart
import 'package:flutter/material.dart';

class AppTheme {
  static const seed = Color(0xFF1F6FEB);

  static ThemeData light() => ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: seed, brightness: Brightness.light),
        textTheme: const TextTheme(
          headlineLarge: TextStyle(fontWeight: FontWeight.w700, fontSize: 28),
          titleLarge: TextStyle(fontWeight: FontWeight.w600, fontSize: 20),
        ),
      );

  static ThemeData dark() => ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: seed, brightness: Brightness.dark),
      );
}
```

Use it in `MaterialApp.router(theme: AppTheme.light(), darkTheme: AppTheme.dark())`.

## 10. Localization (l10n)

Initialize:

```yaml
# app/l10n.yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-class: AppLocalizations
output-localization-file: app_localizations.dart
```

Create `lib/l10n/app_en.arb` with your visible strings. After every
change run `flutter gen-l10n`. Use `AppLocalizations.of(context)!.signIn`
in widgets.

If you ever ship in 2+ languages, add Arabic / Spanish / French ARB
files and use the `omi-add-missing-language-keys-l10n` skill workflow
in our reference repo's `app/CLAUDE.md` as a pattern.

## 11. Flavors (dev / staging / prod)

Use `flutter_flavorizr` or `--dart-define-from-file` to swap
`BASE_API_URL`, `FIREBASE_PROJECT_ID`, app name, and icon per flavor:

```bash
flutter run --dart-define-from-file=env/dev.json
```

Where `env/dev.json` is:

```json
{ "BASE_API_URL": "https://<<YOUR_BRAND>>-dev.ngrok-free.app",
  "ENV": "development" }
```

Read at runtime:

```dart
const baseUrl = String.fromEnvironment('BASE_API_URL');
```

## 12. Releasing

The app stores process is its own beast. The minimum:

- iOS: configure signing in Xcode (Apple Developer account required),
  `flutter build ipa`, upload via Xcode Organizer.
- Android: generate a keystore once,
  `flutter build appbundle --release`, upload to Play Console.
- TestFlight + internal testing tracks first; production release
  later.

For repeatability, use [Codemagic](https://codemagic.io/) or
GitHub Actions with `subosito/flutter-action`.

## 13. Commit

```bash
cd ..
git add app branding
git commit -m "feat(part-21): flutter app shell, branding, listen ws + auth"
git push
```

## What you should have right now

- [ ] `flutter run` shows your app icon and splash with your brand.
- [ ] Sign in with Google + Apple works.
- [ ] Hitting "/me" returns the backend user.
- [ ] Tapping "Listen" opens the WS and streams phone-mic PCM.
- [ ] Live transcripts appear on screen.
- [ ] FCM tokens registered with the backend.
- [ ] App is theme'd, l10n-ready.

---

Next: [Part 22 — Launch Checklist](./22-launch-checklist.md).
