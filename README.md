# SK Calling

Package: `com.skcalling.app`
Firebase App ID: `1:972289125247:android:23734b569354e0a33f484a`

A chat + voice/video calling Android app skeleton:
- **Chat**: text messages, documents (any file type), voice messages, all synced through Firestore + Firebase Storage.
- **Calling**: self-hosted WebRTC (no third-party calling SaaS / no per-minute fees). Firebase Firestore is used only as the *signaling* channel (exchanging SDP offers/answers and ICE candidates) — the actual audio/video always flows peer-to-peer between the two phones.

## 1. Before you open this in Android Studio

Drop your **`google-services.json`** into `app/google-services.json`. Download it from:
Firebase Console → Project Settings → Your apps → the Android app registered as `com.skcalling.app` (App ID `1:972289125247:android:23734b569354e0a33f484a`).

Without this file the Gradle sync will fail on the `com.google.gms.google-services` plugin.

## 2. Firebase products to enable in the console

- **Authentication** → Phone sign-in method (used by `LoginActivity`)
- **Firestore Database** → start in production mode, then add security rules restricting each user to their own chats/calls (see "Security rules" below — the ones shipped in Firebase by default are wide open and NOT safe to ship)
- **Storage** → for documents / voice messages / images
- **Cloud Messaging** → for waking the app to ring on incoming calls and for chat notifications

## 3. TURN server (required for real-world calls)

Two phones on different WiFi/cellular networks usually can't connect with STUN alone. You need a TURN relay:
- Cheapest: run [coturn](https://github.com/coturn/coturn) on a $5/mo VPS
- Or use a managed TURN provider (Twilio, Xirsys, Cloudflare Calls, etc. — usage-based, but you're only using it for TURN relay, not the whole call stack)

Add your TURN URI + credentials in `WebRTCClient.kt` (`iceServers` list — there's a `TODO` marking exactly where).

## 4. Waking a backgrounded/killed app to ring

Firestore listeners don't fire while the app is killed. `CallActivity` has a `TODO` where the caller should trigger a small **Cloud Function**: whenever a new `/calls/{callId}` document is created, the function looks up the callee's `fcmToken` (stored on their `/users/{uid}` doc) and sends a **data-only FCM message** with `type: "incoming_call"`. `FcmService` picks that up and launches `IncomingCallActivity` full-screen, even over the lock screen.

Example Cloud Function (Node.js, add to a separate Firebase Functions project):

```js
exports.onCallCreated = functions.firestore
  .document('calls/{callId}')
  .onCreate(async (snap, context) => {
    const call = snap.data();
    const calleeDoc = await admin.firestore().collection('users').doc(call.calleeId).get();
    const token = calleeDoc.data()?.fcmToken;
    if (!token) return;
    await admin.messaging().send({
      token,
      data: {
        type: 'incoming_call',
        callId: context.params.callId,
        callerId: call.callerId,
        callType: call.type,
      },
      android: { priority: 'high' },
    });
  });
```

## 5. Firestore data model

```
/users/{uid}                     -> displayName, phoneNumber, photoUrl, fcmToken, online
/chats/{chatId}                  -> participants[], lastMessage, lastTimestamp
/chats/{chatId}/messages/{id}    -> senderId, type, text, mediaUrl, fileName, timestamp...
/calls/{callId}                  -> callerId, calleeId, type, state, offerSdp, answerSdp
/calls/{callId}/candidates/{id}  -> sdpMid, sdpMLineIndex, candidate, from
```

`chatId` is deterministic: the two user IDs sorted and joined with `_`, so there's never a
lookup needed to find an existing 1:1 conversation.

## 6. What's stubbed / left for you to finish

- Contact picker on the "new chat" FAB in `MainActivity` (currently a placeholder toast)
- Resolving other users' display names/photos in chat & call headers (currently shows raw uid)
- Image/video capture and thumbnail generation (document + voice message flows are complete)
- Firestore & Storage **security rules** — do not ship with the Firebase defaults
- The Cloud Function above (needs its own `firebase deploy --only functions`)
- Group chats / group calls (this skeleton is 1:1 only)

## 7. Build

Standard Android Studio project. Requires:
- Android Studio Koala+ 
- JDK 17
- `compileSdk 35`, `minSdk 24`

## 8. Building an APK without Android Studio (GitHub Actions)

This repo includes `.github/workflows/build-apk.yml`, which builds a debug APK
on GitHub's servers - no Android SDK, Gradle, or Android Studio install needed
on your own machine.

**One-time setup:**

1. Create a new GitHub repo and push this project to it:
   ```
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/<your-username>/<repo-name>.git
   git push -u origin main
   ```
2. Base64-encode your `google-services.json` so it can be stored as a secret
   instead of committed in plain text:
   ```
   base64 -i google-services.json | tr -d '\n' > google-services.b64.txt
   ```
   (On Windows PowerShell: `[Convert]::ToBase64String([IO.File]::ReadAllBytes("google-services.json")) | Out-File google-services.b64.txt`)
3. In your GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**
   - Name: `GOOGLE_SERVICES_JSON_BASE64`
   - Value: paste the contents of `google-services.b64.txt`
4. Go to the **Actions** tab of your repo → you should see "Build Debug APK" →
   click **Run workflow** (or just push a commit; it also runs automatically on push to `main`).

**Getting the APK onto your phone:**

1. Once the workflow finishes (green check), open that run → scroll to
   **Artifacts** → download `sk-calling-debug-apk` (a zip containing `app-debug.apk`).
2. Transfer the APK to your phone any way you like (email it to yourself,
   upload to Google Drive, or plug in via USB and copy it over).
3. On the phone, tap the APK file. If prompted, allow **"Install unknown apps"**
   for whichever app you opened it with (Files, Chrome, Gmail, etc.) - this is
   an Android security prompt, not something wrong with the build.
4. Tap **Install**. Since Firebase Phone Auth needs SHA-1/SHA-256 fingerprints
   registered in the Firebase console, also add your signing key's fingerprint
   there (for a debug build, run `keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android` locally, or extract it from the Actions log if you add a step for it) under
   Firebase Console → Project Settings → Your app → Add fingerprint.

This produces a **debug** build (unsigned for release, fine for personal testing).
For a Play Store release build you'd extend the workflow with a signing step
and `assembleRelease` instead - ask if you want that added.
