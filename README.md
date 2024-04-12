# Expo-Modules_implementation
A detailed guide for implementing Expo-modules enabling autolinking in an EXISTING Native Andoid Java/Kotlin project

I've put together a guide on integrating React Native Expo into existing Android projects, based on my own experience. While Expo's official documentation covers this topic, I found it somewhat limited and unclear, especially when troubleshooting issues.

Initially, I attempted the automatic installation method using 'npx install-expo-modules@latest', but encountered repeated failures. Then, I turned to the manual installation approach, which provided some insights but didn't fully enable Expo's autolinking feature. Through trial and error, I pieced together the necessary steps, essentially following Expo's manual installation guide but a slightly different approach and outcome, especially useful for those unfamiliar with the Android project structure like myself.
Note that there is still a difference between the Expo's manual installation and my manual installation being the following:
By following my approach the rendering of React Native bundles will only work at the places where a reactRootView is places and configured to render the correct bundle exported from React Native (AppRegistry).
Advantage: having more control over where React Native bundles are rendered relative to (within) the original native app which is useful to keep using the native (existing) navigation etc.
Disadvantage: I suppose it's still less "automatic" and might be more difficult to make two seperate React Native bundles communicate

Expo's manual installation approach, I believe, aims to make 'MainActivity' and 'MainApplication' to be the main and only entry file where your react native bundle is rendered in, thus allowing you to have ONE single NATIVE screen in which the React Native bundle is rendered and navigation then be handled, automatically, from React Native Expo-compatible navigation methods.
Advantage: communcation between all React Native code is guarenteed (since there's only one bundle being rendered)
Disadvantage: it's less relevant for a scenario where an older, completely native, android project is required to remain as is and expand that application by, from then on, writing new React Native screens/components and rendering them as bundles in those existing native screens wherever its necessary

If you're interested, you can check out the official documentation here: Expo Manual Installation Guide.

Or follow my take on the Manual Installation:

State of the my system and the project I have written, applied and used this guide for:
System:
  OS: Windows 11 10.0.22631
  CPU: (8) x64 Intel(R) Core(TM) i5-8265U CPU @ 1.60GHz
Binaries:
  Node: 20.11.1
  Yarn: 1.22.22
  npm: 10.5.0
SDKs:
  Android SDK Platform:
  minSdk: 23
  targetSdkVersion: 34
IDEs:
  Android Studio: AI-232.10300.40.2321.11668458
  WebStorm: 17.0.10+8-b1207.12
Languages:
  Java: 17.0.10
npmPackages:
  react:
    installed: 18.2.0
  react-native: 0.73.6
Android:
  hermesEnabled: true
  newArchEnabled: false

Prerequisites: 
1) Create a normal, standalone, react native project (does not have to be Expo Managed (yet) but it's the same approach):
https://docs.expo.dev/get-started/create-a-project/ (`npx create-expo-app`)
or
https://reactnative.dev/docs/environment-setup (`npx react-native@latest init`)

2) Create a directory 'android' in the root of the React Native project and copy the complete Native Android project into the 'android' directory.

3) Follow the React Native guide 'Integration with Existing Apps' to make the native project correctly set up by installing the necessary React Native dependencies (gradle) and so on: https://reactnative.dev/docs/integration-with-existing-apps?language=java

Preconditioning Expo Conversion in React Native with Original Native Project:
1) Ensure consistent package/bundle ID across the entire project, including both the native and React Native Expo configurations in app.json.

2) Begin by creating/modifying 'MainActivity.kt'; The following bits of code are the essential, minimum, required 'MainActivity' modifications:
Essential imports:
```bash
import android.content.pm.PackageManager
import android.os.Bundle
import android.view.KeyEvent
import android.widget.LinearLayout
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import com.facebook.react.BuildConfig
import com.facebook.react.PackageList
import com.facebook.react.ReactInstanceManager
import com.facebook.react.ReactPackage
import com.facebook.react.ReactRootView
import com.facebook.react.common.LifecycleState
import com.facebook.soloader.SoLoader
import com.facebook.react.modules.core.DefaultHardwareBackBtnHandler
import com.facebook.react.modules.core.PermissionAwareActivity
import com.facebook.react.modules.core.PermissionListener
```

The MainActivity class must extend AppCompatActivity(), DefaultHardwareBackBtnHandler and PermissionAwareActivity:
```bash
class MainActivity : AppCompatActivity(), DefaultHardwareBackBtnHandler, PermissionAwareActivity {
	//MainActivity content
}
```

Essential properties of 'MainActivity' class:
```bash
private lateinit var reactRootView: ReactRootView
private lateinit var reactInstanceManager: ReactInstanceManager
```

The actual React Instance (bundle) declaration and render (onCreate method):
```bash
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        SoLoader.init(this, false)
        reactRootView = ReactRootView(this)
		val reactPackages: List<ReactPackage> = PackageList(application).packages
		
		reactInstanceManager = ReactInstanceManager.builder()
            .setApplication(application)
            .setCurrentActivity(this)
            .setBundleAssetName("index.android.bundle")
            .setJSMainModulePath("index")
            .addPackages(reactPackages)
            .setUseDeveloperSupport(BuildConfig.DEBUG)
            .setInitialLifecycleState(LifecycleState.RESUMED)
            .build()
			
		reactRootView.startReactApplication(reactInstanceManager, "MyReactNativeApp2", null)
		val layout = createLinearLayout()
		layout.addView(reactRootView)
		setContentView(layout)
    }
	
```
	
Essential navigation handler (override) methods:
```bash
override fun invokeDefaultOnBackPressed() {
        super.onBackPressed()
    }

    @Deprecated("Deprecated in Java")
    override fun onBackPressed() {
        reactInstanceManager.onBackPressed()
        super.onBackPressed()
    }

    override fun onPause() {
        super.onPause()
        reactInstanceManager.onHostPause(this)
    }

    override fun onResume() {
        super.onResume()
        reactInstanceManager.onHostResume(this, this)
    }

    override fun onDestroy() {
        super.onDestroy()
        reactInstanceManager.onHostDestroy(this)
        reactRootView.unmountReactApplication()
    }

    override fun onKeyUp(keyCode: Int, event: KeyEvent?): Boolean {
        if (keyCode == KeyEvent.KEYCODE_MENU) {
            reactInstanceManager.showDevOptionsDialog()
            return true
        }
        return super.onKeyUp(keyCode, event)
    }
```

3) Install Expo in the React Native project by running `npm install expo`
4) Run `npx expo install --fix` to ensure proper installation.

5) Create a new file in the same directory as 'MainActivity.kt', named 'MainApplication.kt', and paste the required code which I will upload an example of within this repository (`MainApplication.kt`)

6) Add the following scripts in 'package.json' for a faster workflow:
```bash
"scripts": {
    "dev": "yarn react-native start",
    "android": "yarn react-native run-android",
    "ios": "yarn react-native run-ios",
    "bundle-android": "npx react-native bundle --platform android --dev false --entry-file index.tsx --bundle-output android\\app\\src\\main\\assets\\index.android.bundle --assets-dest android\\app\\src\\main\\res",
    "bundle-ios": "npx react-native bundle --platform ios --dev false --entry-file index.tsx --bundle-output ios\\main.jsbundle --assets-dest ios",
    "bundle": "npm run bundle-android && npm run bundle-ios",
    "build": "npm run bundle-android && npm run bundle-ios",
    "all": "npm run bundle-android && npm run android && npm run bundle-ios && npm run ios"
  }
```
Caution:
Rename the bundle name 'index.android.bundle' to whatever you want it to be named if you are going to make use of multiple bundles
Rename the entry file 'index.tsx' to whatever your React Native entry file is called and also pay attention to the file extension; '.tsx' for TypeScript and '.js' for JavaScript.

7) Ensure the following properties are enabled in 'gradle.properties' (project properties):
```bash
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
android.nonTransitiveRClass=true
android.enableJetifier=true
hermesEnabled=true
android.nonFinalResIds=false
EX_DEV_CLIENT_NETWORK_INSPECTOR=true
expo.useLegacyPackaging=false
```

8) Confirm that necessary dependencies are present in 'settings.gradle' (project settings):
```bash
include ':app'
includeBuild('../node_modules/@react-native/gradle-plugin')
apply from: file("../node_modules/@react-native-community/cli-platform-android/native_modules.gradle");
applyNativeModulesSettingsGradle(settings)
apply from: new File(["node", "--print", "require.resolve('expo/package.json')"].execute(null, rootDir).text.trim(), "../scripts/autolinking.gradle")
useExpoModules()
```

9) Double check the top-level 'build.gradle' file; it should already contain the following 'buildscript', 'plugins' and 'allprojects' blocks:
```bash
buildscript {
    repositories {
        jcenter()
        mavenCentral()
        google()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.3.1'
        classpath("com.facebook.react:react-native-gradle-plugin")
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
plugins {
    id "org.jetbrains.kotlin.android" version "1.9.0" apply false
}
allprojects {
    repositories {
        jcenter()
        mavenCentral()
        google()
        maven { url "https://jitpack.io" }

        // Add repositories from Kotlin project
        maven { url "https://dl.bintray.com/journeyapps/maven" }
    }
}
```
Note: at the bottom of this file should also be a task declaration responsable for correctly cleaning builds:
```bash
tasks.register('clean', Delete) {
    delete rootProject.buildDir
}
```

10) When the original native project is written in Java, make sure to apply plugin 'kotlin-android' at the top of the app-level build.gradle where at this points; at least two other plugins should already be present:
```bash
apply plugin: 'com.android.application'
apply plugin: "com.facebook.react"
apply plugin: 'kotlin-android'
//rest of the code
```
Note: at the bottom of the (app-level) 'build.gradle' file should already be the following code, configured from the React Native 'Integration with Existing Apps' guide:
```bash
//rest of the code
apply plugin: 'com.android.application'
apply plugin: "com.facebook.react"
apply plugin: 'kotlin-android'
```

11) Add the following activity declaration in 'AndroidManifest.xml':
```bash
<activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />
```

These steps were necessary for my OLD, Java Android, project in order to make integrating of Expo Modules into your existing Android project possible; making the development process better since there will be no need for native configuration as long as Expo-compatible packages are used in React Native. Keep in mind that the new entry files for Expo modules integration will be in Kotlin. However, for the native Java project, there's no need to convert syntax since Java and Kotlin can coexist in the same project, albeit not in the same files.

NOTE: I have very minimal experience working with Java for mobile android apps and thus I could be wrong about specific "claims" made in my above findings, but I can proudly say that I am now able to write react native code, keep practicing the expo managed workflow and never have to touch any native code (apart from this one-time configuration and the places I want my React Native bundles to be rendered), to make the react native components appear in development and render in production builds WITHIN the original, Java-written, native application.
