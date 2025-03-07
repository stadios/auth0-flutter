# Frequently Asked Questions

1. [How can I have separate Auth0 domains for each environment on Android?](#1-how-can-i-have-separate-auth0-domains-for-each-environment-on-android)
2. [How can I disable the iOS _login_ alert box?](#2-how-can-i-disable-the-ios-login-alert-box)
3. [How can I disable the iOS _logout_ alert box?](#3-how-can-i-disable-the-ios-logout-alert-box)
4. [Is there a way to disable the iOS _login_ alert box without `useEphemeralSession`?](#4-is-there-a-way-to-disable-the-ios-login-alert-box-without-useephemeralsession)
5. [How can I change the message in the iOS alert box?](#5-how-can-i-change-the-message-in-the-ios-alert-box)
6. [How can I programmatically close the iOS alert box?](#6-how-can-i-programmatically-close-the-ios-alert-box)

## 1. How can I have separate Auth0 domains for each environment on Android?

This library uses [Auth0.Android](https://github.com/auth0/Auth0.Android) under the hood on Android. Auth0.Android declares a `RedirectActivity` along with an **intent-filter** in its Android Manifest file to handle the Web Auth callback and logout URLs. While this approach prevents the developer from adding an activity declaration to their app's Android Manifest file, it requires the use of [manifest placeholders](https://developer.android.com/studio/build/manage-manifests#inject_build_variables_into_the_manifest).

Alternatively, you can re-declare the `RedirectActivity` in the `android/app/src/main/AndroidManifest.xml` file with your own **intent-filter** so it overrides the library's default one. If you do this then the `manifestPlaceholders` don't need to be set as long as the activity contains `tools:node="replace"` like in the snippet below.

```xml
<!-- android/src/main/AndroidManifest.xml -->

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.company.myapp">
    <application android:theme="@style/AppTheme">
        <!-- ... -->

        <activity
            android:name="com.auth0.react.RedirectActivity"
            tools:node="replace">
            <intent-filter
                android:autoVerify="true"
                tools:targetApi="m">
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <!-- add a data tag for each environment -->
                <data
                    android:host="example.com"
                    android:pathPrefix="/android/${applicationId}/callback"
                    android:scheme="${auth0Scheme}" />
                <data
                    android:host="qa.example.com"
                    android:pathPrefix="/android/${applicationId}/callback"
                    android:scheme="${auth0Scheme}" />
            </intent-filter>
        </activity>

        <!-- ... -->
    </application>
</manifest>
```

## 2. How can I disable the iOS _login_ alert box?

![ios-sso-alert](assets/ios-sso-alert.png)

This library uses [Auth0.swift](https://github.com/auth0/Auth0.swift) under the hood on iOS. Auth0.swift uses `ASWebAuthenticationSession` to perform web-based authentication, which is the [API provided by Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession) for such purpose.

That alert box is displayed and managed by `ASWebAuthenticationSession`, not by Auth0.swift, because by default this API will store the session cookie in the shared Safari cookie jar. This makes Single Sign-On (SSO) possible. According to Apple, that requires user consent.

> 💡 See [this blog post](https://developer.okta.com/blog/2022/01/13/mobile-sso) for a detailed overview of SSO on iOS.

If you don't need SSO, you can disable this behavior by adding `useEphemeralSession: true` to the login call. This will configure `ASWebAuthenticationSession` to not store the session cookie in the shared cookie jar, as if using an incognito browser window. With no shared cookie, `ASWebAuthenticationSession` will not prompt the user for consent.

```dart
final credentials = await auth0
    .webAuthentication()
    .login(useEphemeralSession: true); // No SSO, therefore no alert box
```

Note that with `useEphemeralSession: true` you don't need to call `logout()` at all. Just clearing the credentials from the app will suffice. What `logout()` does is clear the shared session cookie, so that in the next login call the user gets asked to log in again. But with `useEphemeralSession: true` there will be no shared cookie to remove.

You still need to call `logout()` on Android, though, as `useEphemeralSession` is iOS-only.

> ⚠️ `useEphemeralSession` relies on the `prefersEphemeralWebBrowserSession` configuration option of `ASWebAuthenticationSession`. This option is only available on [iOS 13+](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/3237231-prefersephemeralwebbrowsersessio), so `useEphemeralSession` will have no effect on iOS 12. To improve the experience for iOS 12 users, see the approach described below.

## 3. How can I disable the iOS _logout_ alert box?

![ios-sso-alert](assets/ios-sso-alert.png)

If you need SSO and/or are willing to tolerate the alert box on the login call, but would prefer to get rid of it when calling `logout()`, you can simply not call `logout()` and just clear the credentials from the app. This means that the shared session cookie will not be removed, so to get the user to log in again you need to add the `'prompt': 'login'` parameter to the _login_ call.

```dart
final credentials = await auth0.webAuthentication().login(
    useEphemeralSession: true,
    parameters: {
      'prompt': 'login'
    }); // Ignore the cookie (if present) and show the login page
```

Otherwise, the browser modal will close right away and the user will be automatically logged in again, as the cookie will still be there.

> ⚠️ Keeping the shared session cookie may not be an option if you have strong privacy and/or security requirements, for example in the case of a banking app.

## 4. Is there a way to disable the iOS _login_ alert box without `useEphemeralSession`?

No. According to Apple, storing the session cookie in the shared Safari cookie jar requires user consent. The only way to not have a shared cookie is to configure `ASWebAuthenticationSession` with `prefersEphemeralWebBrowserSession` set to `true`, which is what `useEphemeralSession: true` does.

## 5. How can I change the message in the iOS alert box?

This library has no control whatsoever over the alert box. Its contents cannot be changed. Unfortunately, that's a limitation of `ASWebAuthenticationSession`.

## 6. How can I programmatically close the iOS alert box?

This library has no control whatsoever over the alert box. It cannot be closed programmatically. Unfortunately, that's a limitation of `ASWebAuthenticationSession`.
