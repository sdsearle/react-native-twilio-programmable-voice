# react-native-twilio-programmable-voice

This is a react-native wrapper around [Twilio Programmable Voice SDK](https://www.twilio.com/voice), which allows your react-native app to make and receive calls.

This module is not affiliated with nor officially maintained by Twilio. It is maintained by open source contributors working during their nights and weekends.

Tested with:

- react-native 0.62.2
- Android 11
- iOS 14

## Roadmap

### Project 1

The most updated branch is [feat/twilio-android-sdk-5](https://github.com/hoxfon/react-native-twilio-programmable-voice/tree/feat/twilio-android-sdk-5) which is aligned with:

- Android 5.4.2
- iOS 5.2.0

It contains breaking changes from `react-native-twilio-programmable-voice` v4, and it will be released as v5.

You can install it with:

```bash
# Yarn
yarn add https://github.com/hoxfon/react-native-twilio-programmable-voice#feat/twilio-android-sdk-5

# NPM
npm install git+https://github.com/hoxfon/react-native-twilio-programmable-voice#feat/twilio-android-sdk-5
```

I am currently updating the library to catchup with all changes published on the latest Android and iOS Twilio Voice SDK:

[iOS changelog](https://www.twilio.com/docs/voice/voip-sdk/ios/changelog)
[Android changelog](https://www.twilio.com/docs/voice/voip-sdk/android/3x-changelog)

My plan is to use the following links as a reference, and follow Twilio commit by commit.

- https://github.com/twilio/voice-quickstart-android
- https://github.com/twilio/voice-quickstart-ios

_If you want to contribute please consider helping on this project. [Click here for more information](https://github.com/hoxfon/react-native-twilio-programmable-voice/issues/158)._

### Project 2

Allow Android to use the built in Android telephony service to make and receive calls.

## Stable release

### Twilio Programmable Voice SDK

- Android 4.5.0
- iOS 5.2.0

### Breaking changes in v5.0.0

Changes on [Android Twilio Voice SDK v5](https://www.twilio.com/docs/voice/voip-sdk/android/3x-changelog#500) are reflected in the JavaScript API, the way call invites are handled has changed and other v5 features like `audioSwitch` have been implemented.
`setSpeakerPhone()` has been removed from Android, use selectAudioDevice(name: string) instead.

#### Background incoming calls

- When the app is not in foreground incoming calls result in a heads-up notification with action to "ACCEPT" and "REJECT".
- ReactMethod `accept` does not dispatch any event. In v4 it dispatched `connectionDidDisconnect`.
- ReactMethod `reject` dispatches a `callInviteCancelled` event instead of `connectionDidDisconnect`.
- ReactMethod `ignore` does not dispatch any event. In v4 it dispatched `connectionDidDisconnect`.

To show heads up notifications, you must add the following lines to your application's `android/app/src/main/AndroidManifest.xml`:

```xml
    <!-- receive calls when the app is in the background-->
    <uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

    <application
        ...
    >
        <!-- Twilio Voice -->
        <!-- [START fcm_listener] -->
        <service
            android:name="com.hoxfon.react.RNTwilioVoice.fcm.VoiceFirebaseMessagingService"
            android:stopWithTask="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>
        <service
            android:enabled="true"
            android:name="com.hoxfon.react.RNTwilioVoice.IncomingCallNotificationService"
            android:foregroundServiceType="phoneCall">
            <intent-filter>
                <action android:name="com.hoxfon.react.RNTwilioVoice.ACTION_ACCEPT" />
                <action android:name="com.hoxfon.react.RNTwilioVoice.ACTION_REJECT" />
            </intent-filter>
        </service>
        <!-- [END fcm_listener] -->
        <!-- Twilio Voice -->
    </application>
```

Firebase Messaging 19.0.+ is imported by this module, so there is no need to import it in your app's `bundle.gradle` file.

In v4 the flow to launch the app when receiving a call was:

1. the module launched the app
2. after the React app is initialised, it always asked to the native module whether there were incoming call invites
3. if there were any incoming call invites, the module would have sent an event to the React app with the incoming call invite parameters
4. the Reach app would have listened to the event and would have launched the view with the appropriate incoming call answer/reject controls

This loop was long and prone to race conditions. For example,when the event was sent before the React main view was completely initialised, it would not be handled at all.

V5 replaces the previous flow by using `getLaunchOptions()` to pass initial properties from the native module to React, when receiving a call invite as explained here: https://reactnative.dev/docs/communication-android.

The React app is launched with the initial properties `callInvite` or `call`.

To handle correctly `lauchedOptions`, you must add the following blocks to your app's `MainActivity`:

```java

import com.hoxfon.react.RNTwilioVoice.TwilioModule;
...

public class MainActivity extends ReactActivity {

    @Override
    protected ReactActivityDelegate createReactActivityDelegate() {
        return new ReactActivityDelegate(this, getMainComponentName()) {
            @Override
            protected ReactRootView createRootView() {
                return new RNGestureHandlerEnabledRootView(MainActivity.this);
            }
            @Override
            protected Bundle getLaunchOptions() {
                return TwilioModule.getActivityLaunchOption(this.getPlainActivity().getIntent());
            }
        };
    }

    // ...

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O_MR1) {
            setShowWhenLocked(true);
            setTurnScreenOn(true);
        }
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                | WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON | WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                | WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);
    }

    // ...
}
```

#### Audio Switch

Access to native Twilio SDK AudioSwitch module for Android has been added to the JavaScript API:

```javascript
// getAudioDevices returns all audio devices connected
// {
//     "Speakerphone": false,
//     "Earnpiece": true, // true indicates the selected device
// }
getAudioDevices()

// getSelectedAudioDevice returns the selected audio device
getSelectedAudioDevice()

// selectAudioDevice selects the passed audio device for the current active call
selectAudioDevice(name: string)
```

#### Event deviceDidReceiveIncoming

When a call invite is received, the [SHAKEN/STIR](https://www.twilio.com/docs/voice/trusted-calling-using-shakenstir) `caller_verification` field has been added to the list of params for  `deviceDidReceiveIncoming`. Values are: `verified`, `unverified`, `unknown`.

## ICE

See https://www.twilio.com/docs/stun-turn

```bash
curl -X POST https://api.twilio.com/2010-04-01/Accounts/ACb0b56ae3bf07ce4045620249c3c90b40/Tokens.json \
-u ACb0b56ae3bf07ce4045620249c3c90b40:f5c84f06e5c02b55fa61696244a17c84
```

```java
Set<IceServer> iceServers = new HashSet<>();
// server URLs returned by calling the Twilio Rest API to generate a new token
iceServers.add(new IceServer("stun:global.stun.twilio.com:3478?transport=udp"));
iceServers.add(new IceServer("turn:global.turn.twilio.com:3478?transport=udp","8e6467be547b969ad913f7bdcfb73e411b35f648bd19f2c1cb4161b4d4a067be","n8zwmkgjIOphHN93L/aQxnkUp1xJwrZVLKc/RXL0ZpM="));
iceServers.add(new IceServer("turn:global.turn.twilio.com:3478?transport=tcp","8e6467be547b969ad913f7bdcfb73e411b35f648bd19f2c1cb4161b4d4a067be","n8zwmkgjIOphHN93L/aQxnkUp1xJwrZVLKc/RXL0ZpM="));
iceServers.add(new IceServer("turn:global.turn.twilio.com:443?transport=tcp","8e6467be547b969ad913f7bdcfb73e411b35f648bd19f2c1cb4161b4d4a067be","n8zwmkgjIOphHN93L/aQxnkUp1xJwrZVLKc/RXL0ZpM="));

IceOptions iceOptions = new IceOptions.Builder()
		.iceServers(iceServers)
		.build();

ConnectOptions connectOptions = new ConnectOptions.Builder(accessToken)
		.iceOptions(iceOptions)
		.enableDscp(true)
		.params(twiMLParams)
		.build();
```

### Breaking changes in v4.0.0

The module implements [react-native autolinking](https://github.com/react-native-community/cli/blob/master/docs/autolinking.md) as many other native libraries > react-native 0.60.0, therefore it doesn't need to be linked manually.

Android: update Firebase Messaging to 17.6.+. Remove the following block from your application's `AndroidManifest.xml` if you are migrating from v3.

```xml
    <!-- [START instanceId_listener] -->
    <service
        android:name="com.hoxfon.react.TwilioVoice.fcm.VoiceFirebaseInstanceIDService"
        android:exported="false">
        <intent-filter>
            <action android:name="com.google.android.gms.iid.InstanceID" />
        </intent-filter>
    </service>
    <!-- [END instanceId_listener] -->
```

Android X is supported.

Data passed to the event `deviceDidReceiveIncoming` does not contain the key `call_state`, because state of Call Invites was removed in Twilio Android and iOS SDK v3.0.0

- iOS: params changes for `connectionDidConnect` and `connectionDidDisconnect`

    to => call_to
    from => call_from
    error => err

New features

Twilio Programmable Voice SDK v3.0.0 handles call invites directly and makes it easy to distinguish a call invites from an active call, which previously was confusing.
To ensure that an active call is displayed when the app comes to foreground you should use the promise `getActiveCall()`.
To ensure that a call invite is displayed when the app comes to foreground use the promise `getCallInvite()`. Please note that call invites don't have a `call_state` field.

You should use `hold()` to put a call on hold.

You can be notified when a call is `ringing` by listening for `callStateRinging` events.

iOS application can now receive the following events, that in v3 where only dispatched to Android:

- deviceDidReceiveIncoming
- callInviteCancelled
- callStateRinging
- connectionIsReconnecting
- connectionDidReconnect

### Breaking changes in v3.0.0

- initWitToken returns an object with a property `initialized` instead of `initilized`
- iOS event `connectionDidConnect` returns the same properties as Android
move property `to` => `call_to`
move property `from` => `call_from`

### Installation

Before starting, we recommend you get familiar with [Twilio Programmable Voice SDK](https://www.twilio.com/docs/api/voice-sdk).
It's easier to integrate this module into your react-native app if you follow the Quick start tutorial from Twilio, because it makes very clear which setup steps are required.

```bash
npm install react-native-twilio-programmable-voice --save
```

- **React Native 0.60+**

[CLI autolink feature](https://github.com/react-native-community/cli/blob/master/docs/autolinking.md) links the module while building the app.

- **React Native <= 0.59**

```bash
react-native link react-native-twilio-programmable-voice
```

### iOS Installation

If you can't or don't want to use autolink, you can also manually link the library using the instructions below (click on the arrow to show them):

<details>
<summary>Manually link the library on iOS</summary>

Follow the [instructions in the React Native documentation](https://facebook.github.io/react-native/docs/linking-libraries-ios#manual-linking) to manually link the framework

After you have linked the library with `react-native link react-native-twilio-programmable-voice`
check that `libRNTwilioVoice.a` is present under YOUR_TARGET > Build Phases > Link Binaries With Libraries. If it is not present you can add it using the + sign at the bottom of that list.
</details>

```bash
cd ios && pod install
```

#### CallKit

The iOS library works through [CallKit](https://developer.apple.com/reference/callkit) and handling calls is much simpler than the  Android implementation as CallKit handles the inbound calls answering, ignoring, or rejecting. Outbound calls must be controlled by custom React-Native screens and controls.

To pass caller's name to CallKit via Voip push notification add custom parameter 'CallerName' to Twilio Dial verb.

```xml
    <Dial>
    <Client>
        <Identity>Client</Identity>
        <Parameter name="CallerName">NAME TO DISPLAY</Parameter>
    </Client>
    </Dial>
```

#### VoIP Service Certificate

Twilio Programmable Voice for iOS utilizes Apple's VoIP Services and VoIP "Push Notifications" instead of FCM. You will need a VoIP Service Certificate from Apple to receive calls. Follow [the official Twilio instructions](https://github.com/twilio/voice-quickstart-ios#7-create-voip-service-certificate) to complete this step.

### Android Installation

Setup FCM

You must download the file `google-services.json` from the Firebase console.
It contains keys and settings for all your applications under Firebase. This library obtains the resource `senderID` for registering for remote GCM from that file.

#### `android/build.gradle`

```groovy
buildscript {
    dependencies {
        // override the google-service version if needed
        // https://developers.google.com/android/guides/google-services-plugin
        classpath 'com.google.gms:google-services:4.3.4'
    }
}

// this plugin looks for google-services.json in your project
apply plugin: 'com.google.gms.google-services'
```

#### `AndroidManifest.xml`

```xml
    <uses-permission android:name="android.permission.VIBRATE" />

    <application ....>
        <!-- Twilio Voice -->
        <!-- [START fcm_listener] -->
        <service
            android:name="com.hoxfon.react.RNTwilioVoice.fcm.VoiceFirebaseMessagingService"
            android:stopWithTask="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>
        <!-- [END fcm_listener] -->
```

If you can't or don't want to use autolink, you can also manually link the library using the instructions below (click on the arrow to show them):

<details>
<summary>Manually link the library on Android</summary>

Make the following changes:

#### `android/settings.gradle`

```groovy
include ':react-native-twilio-programmable-voice'
project(':react-native-twilio-programmable-voice').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-twilio-programmable-voice/android')
```

#### `android/app/build.gradle`

```groovy
dependencies {
   implementation project(':react-native-twilio-programmable-voice')
}
```

#### `android/app/src/main/.../MainApplication.java`
On top, where imports are:

```java
import com.hoxfon.react.RNTwilioVoice.TwilioVoicePackage;  // <--- Import Package
```

Add the `TwilioVoicePackage` class to your list of exported packages.

```java
@Override
protected List<ReactPackage> getPackages() {
    return Arrays.asList(
            new MainReactPackage(),
            new TwilioVoicePackage()         // <---- Add the package
            // new TwilioVoicePackage(false) // <---- pass false if you don't want to ask for microphone permissions
    );
}
```
</details>

## Usage

```javascript
import TwilioVoice from 'react-native-twilio-programmable-voice'

// ...

// initialize the Programmable Voice SDK passing an access token obtained from the server.
// Listen to deviceReady and deviceNotReady events to see whether the initialization succeeded.
async function initTelephony() {
    try {
        const accessToken = await getAccessTokenFromServer()
        const success = await TwilioVoice.initWithToken(accessToken)
    } catch (err) {
        console.err(err)
    }
}

function initTelephonyWithToken(token) {
    TwilioVoice.initWithAccessToken(token)

    // iOS only, configure CallKit
    try {
        TwilioVoice.configureCallKit({
            appName:       'TwilioVoiceExample',                  // Required param
            imageName:     'my_image_name_in_bundle',             // OPTIONAL
            ringtoneSound: 'my_ringtone_sound_filename_in_bundle' // OPTIONAL
        })
    } catch (err) {
        console.err(err)
    }
}
```

## Events

```javascript
// add listeners (flowtype notation)
TwilioVoice.addEventListener('deviceReady', function() {
    // no data
})
TwilioVoice.addEventListener('deviceNotReady', function(data) {
    // {
    //     err: string
    // }
})
TwilioVoice.addEventListener('connectionDidConnect', function(data) {
    // {
    //     call_sid: string,  // Twilio call sid
    //     call_state: 'CONNECTED' | 'ACCEPTED' | 'CONNECTING' | 'RINGING' | 'DISCONNECTED' | 'CANCELLED',
    //     call_from: string, // "+441234567890"
    //     call_to: string,   // "client:bob"
    // }
})
TwilioVoice.addEventListener('connectionIsReconnecting', function(data) {
    // {
    //     call_sid: string,  // Twilio call sid
    //     call_from: string, // "+441234567890"
    //     call_to: string,   // "client:bob"
    // }
})
TwilioVoice.addEventListener('connectionDidReconnect', function(data) {
    // {
    //     call_sid: string,  // Twilio call sid
    //     call_from: string, // "+441234567890"
    //     call_to: string,   // "client:bob"
    // }
})
TwilioVoice.addEventListener('connectionDidDisconnect', function(data: mixed) {
    //   | null
    //   | {
    //       err: string
    //     }
    //   | {
    //         call_sid: string,  // Twilio call sid
    //         call_state: 'CONNECTED' | 'ACCEPTED' | 'CONNECTING' | 'RINGING' | 'DISCONNECTED' | 'CANCELLED',
    //         call_from: string, // "+441234567890"
    //         call_to: string,   // "client:bob"
    //         err?: string,
    //     }
})
TwilioVoice.addEventListener('callStateRinging', function(data: mixed) {
    //   {
    //       call_sid: string,  // Twilio call sid
    //       call_state: 'CONNECTED' | 'ACCEPTED' | 'CONNECTING' | 'RINGING' | 'DISCONNECTED' | 'CANCELLED',
    //       call_from: string, // "+441234567890"
    //       call_to: string,   // "client:bob"
    //   }
})
TwilioVoice.addEventListener('callInviteCancelled', function(data: mixed) {
    //   {
    //       call_sid: string,  // Twilio call sid
    //       call_from: string, // "+441234567890"
    //       call_to: string,   // "client:bob"
    //   }
})

// iOS Only
TwilioVoice.addEventListener('callRejected', function(value: 'callRejected') {})

TwilioVoice.addEventListener('deviceDidReceiveIncoming', function(data) {
    // {
    //     call_sid: string,  // Twilio call sid
    //     call_from: string, // "+441234567890"
    //     call_to: string,   // "client:bob"
    // }
})

// Android Only
TwilioVoice.addEventListener('proximity', function(data) {
    // {
    //     isNear: boolean
    // }
})

// Android Only
TwilioVoice.addEventListener('wiredHeadset', function(data) {
    // {
    //     isPlugged: boolean,
    //     hasMic: boolean,
    //     deviceName: string
    // }
})

// ...

// start a call
TwilioVoice.connect({To: '+61234567890'})

// hangup
TwilioVoice.disconnect()

// accept an incoming call (Android only, in iOS CallKit provides the UI for this)
TwilioVoice.accept()

// reject an incoming call (Android only, in iOS CallKit provides the UI for this)
TwilioVoice.reject()

// ignore an incoming call (Android only)
TwilioVoice.ignore()

// mute or un-mute the call
// mutedValue must be a boolean
TwilioVoice.setMuted(mutedValue)

// put a call on hold
TwilioVoice.hold(holdValue)

// send digits
TwilioVoice.sendDigits(digits)

// Ensure that an active call is displayed when the app comes to foreground
TwilioVoice.getActiveCall()
    .then(activeCall => {
        if (activeCall){
            _displayActiveCall(activeCall)
        }
    })

// Ensure that call invites are displayed when the app comes to foreground
TwilioVoice.getCallInvite()
    .then(callInvite => {
        if (callInvite){
            _handleCallInvite(callInvite)
        }
    })

// Unregister device with Twilio
TwilioVoice.unregister()
```

## Help wanted

There is no need to ask permissions to contribute. Just open an issue or provide a PR. Everybody is welcome to contribute.

ReactNative success is directly linked to its module ecosystem. One way to make an impact is helping contributing to this module or another of the many community lead ones.

![help wanted](images/vjeux_tweet.png "help wanted")

## Twilio Voice SDK reference

[iOS changelog](https://www.twilio.com/docs/voice/voip-sdk/ios/changelog)
[Android changelog](https://www.twilio.com/docs/voice/voip-sdk/android/3x-changelog)

## Credits

[voice-quickstart-android](https://github.com/twilio/voice-quickstart-android)

[voice-quickstart-ios](https://github.com/twilio/voice-quickstart-ios)

[react-native-push-notification](https://github.com/zo0r/react-native-push-notification)

## License

MIT
